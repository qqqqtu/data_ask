# 智能问数 Data Agent

基于 **LangGraph + RAG** 的智能问数系统：用户用自然语言提问，系统自动检索元数据知识库、生成并校验 SQL、执行后流式返回查询结果。

前端提供类 ChatGPT 的对话界面，实时展示 Agent 的每一步执行进度。

---

## 核心特性

- **自然语言转 SQL**：将口语化问题（如"统计去年各地区的销售总额"）自动转换为可执行 SQL
- **RAG 元数据召回**：基于向量检索 + 全文检索，从元数据知识库中召回相关的表、列、指标、字段取值
- **多路并行召回**：列召回（Qdrant）、取值召回（Elasticsearch）、指标召回（Qdrant）三路并发，再合并去重
- **LLM 二次过滤**：用大模型对召回的表和指标做相关性过滤，剔除噪声，节省上下文
- **SQL 校验与自动纠错**：校验生成的 SQL，若报错则进入纠错节点重新生成，提升成功率
- **SSE 流式输出**：通过 Server-Sent Events 实时推送每个节点的执行进度与最终结果

---

## 系统架构

### Agent 工作流（LangGraph）

```
                        START
                          │
                  extract_keywords          # 用 LLM 从问题中抽取关键词
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │        # 三路并行召回
   recall_column     recall_value     recall_metric
   (Qdrant 向量)     (ES 全文检索)     (Qdrant 向量)
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                merge_retrieved_info        # 合并多路召回结果并去重
                          │
                ┌─────────┴─────────┐
          filter_table          filter_metric   # LLM 过滤无关表 / 指标
                └─────────┬─────────┘
                          │
                 add_extra_context          # 补充字段取值、示例等上下文
                          │
                   generate_sql             # LLM 生成 SQL
                          │
                    validate_sql            # 校验 SQL
                          │
              ┌───────────┴───────────┐
        error is None             error is not None
              │                       │
         execute_sql  ◄──────────  correct_sql     # 出错则自动纠错后重试
              │
             END
```

### 分层结构

| 目录 | 职责 |
|---|---|
| `app/api` | FastAPI 路由、依赖注入、请求/响应模型 |
| `app/agent` | LangGraph 图定义、状态、上下文、各节点实现 |
| `app/agent/nodes` | 12 个独立节点，每个节点单一职责 |
| `app/services` | 业务服务层（查询编排、元知识库构建） |
| `app/repositories` | 数据访问层，按存储介质划分（mysql / qdrant / es） |
| `app/clients` | 各外部中间件的异步客户端管理器 |
| `app/models` | SQLAlchemy ORM 模型 |
| `app/entities` | 领域实体（DTO） |
| `app/conf` | 基于 OmegaConf 的类型安全配置加载 |
| `app/prompt` | Prompt 模板加载器 |
| `prompts/` | 各节点的 Prompt 文本模板 |
| `conf/` | 应用配置与元数据配置 |
| `docker/` | 中间件编排与初始化 SQL |
| `data-agent-fronted/` | Vue 3 + Vite 对话式前端 |

### 技术栈

**后端**：Python 3.12 · FastAPI · LangGraph · LangChain · SQLAlchemy(async) · OmegaConf · Loguru
**存储/中间件**：MySQL 8.0（元数据 + 数仓）· Qdrant（向量检索）· Elasticsearch 8.19 + IK 分词（取值检索）· Text Embeddings Inference（bge-large-zh-v1.5 向量化）
**前端**：Vue 3 · Vite

---

## 快速开始

### 1. 前置要求

- Python **3.12+**
- [uv](https://docs.astral.sh/uv/)（推荐的包管理器）
- Docker / Docker Compose
- 一个 DeepSeek API Key（或兼容 OpenAI 协议的其他服务）

### 2. 启动中间件

> ⚠️ 需要先准备 Embedding 模型权重，见下方「[模型权重下载](#模型权重下载)」。

```bash
cd docker
docker compose up -d
```

启动的组件：

| 组件 | 地址 | 说明 |
|---|---|---|
| MySQL | `localhost:3306` | 自动执行 `mysql/*.sql` 初始化 `meta`、`dw` 两库 |
| Elasticsearch | `localhost:9200` | 已内置 IK 中文分词插件 |
| Kibana | `localhost:5601` | ES 可视化（可选） |
| Qdrant | `localhost:6333` | 向量数据库 |
| Embedding | `localhost:8081` | 文本向量化服务 |

### 3. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env`，填入你的 API Key：

```dotenv
LLM_API_KEY=sk-your-real-api-key
```

其余配置（数据库地址、端口等）见 [`conf/app_config.yaml`](conf/app_config.yaml)，默认值已与上面的 Docker Compose 对齐。

### 4. 安装依赖

```bash
uv sync
```

### 5. 构建元数据知识库

将 `conf/meta_config.yaml` 中定义的表、列、指标信息写入 MySQL，并把列/指标向量化写入 Qdrant、把字段取值写入 Elasticsearch。

```bash
uv run python -m app.scripts.build_meta_knowledge -c conf/meta_config.yaml
```

### 6. 启动后端

```bash
uv run python main.py
```

服务运行在 <http://localhost:8001>，接口文档见 <http://localhost:8001/docs>。

### 7. 启动前端

```bash
cd data-agent-fronted
npm install
npm run dev
```

打开终端提示的地址（默认 <http://localhost:5173>），即可开始对话问数。

---

## 模型权重下载

出于仓库体积考虑，**Embedding 模型权重未包含在仓库中**。请自行下载 `bge-large-zh-v1.5` 并放到指定目录：

```bash
# 方式一：huggingface-cli
pip install -U "huggingface_hub[cli]"
huggingface-cli download BAAI/bge-large-zh-v1.5 \
  --local-dir docker/embedding/bge-large-zh-v1.5

# 方式二：ModelScope（国内网络更快）
pip install modelscope
modelscope download --model BAAI/bge-large-zh-v1.5 \
  --local_dir docker/embedding/bge-large-zh-v1.5
```

下载完成后目录结构应为：

```
docker/embedding/bge-large-zh-v1.5/
├── config.json
├── pytorch_model.bin        # 约 1.24 GB
├── tokenizer.json
├── tokenizer_config.json
├── vocab.txt
└── ...
```

---

## API 说明

### `POST /api/query`

以 SSE（`text/event-stream`）流式返回 Agent 执行过程与结果。

**请求体**

```json
{ "query": "统计去年各地区的销售总额" }
```

**响应事件类型**

| `type` | 说明 | 示例 |
|---|---|---|
| `progress` | 节点执行进度 | `{"type":"progress","step":"生成SQL","status":"running"}` |
| `result` | 查询结果集 | `{"type":"result","data":[{"region_name":"华东","total":123}]}` |
| `error` | 异常信息 | `{"type":"error","message":"..."}` |

**调用示例**

```bash
curl -N -X POST http://localhost:8001/api/query \
  -H "Content-Type: application/json" \
  -d '{"query":"统计去年各地区的销售总额"}'
```

---

## 配置说明

### `conf/app_config.yaml`

应用运行配置：日志、MySQL（元数据库 `meta` / 数仓 `dw`）、Qdrant、Embedding 服务、Elasticsearch、LLM。

其中 API Key 通过环境变量注入，不落盘：

```yaml
llm:
  model_name: deepseek-chat
  api_key: ${oc.env:LLM_API_KEY}
  base_url: https://api.deepseek.com/v1
```

### `conf/meta_config.yaml`

元数据知识库的定义文件，描述数仓中有哪些表、每张表的列及其类型/角色/别名/是否同步，以及业务指标定义。构建脚本据此生成知识库。

---

## 目录结构

```
.
├── app/
│   ├── agent/            # LangGraph 工作流
│   │   └── nodes/        # 各执行节点
│   ├── api/              # FastAPI 路由
│   ├── clients/          # 中间件客户端管理
│   ├── conf/             # 配置加载
│   ├── core/             # 日志、上下文、生命周期
│   ├── entities/         # 领域实体
│   ├── models/           # ORM 模型
│   ├── prompt/           # Prompt 加载
│   ├── repositories/     # 数据访问层
│   ├── scripts/          # 知识库构建脚本
│   └── services/         # 业务服务
├── conf/                 # 配置文件
├── data-agent-fronted/   # Vue 3 前端
├── docker/               # 中间件编排
├── prompts/              # Prompt 模板
├── main.py               # 应用入口
└── pyproject.toml
```

---

## 说明

- 项目中的数据库账号密码（`Atguigu.123`）仅用于**本地开发**，生产环境请替换为强密码并改用密钥管理服务。
- `.env` 已被 `.gitignore` 忽略，请勿提交真实密钥。
