# 智能问数 · 前端

智能问数 Data Agent 的对话式前端，基于 **Vue 3 + Vite**。

提供类 ChatGPT 的问答界面：输入自然语言问题后，通过 SSE 实时接收并展示 Agent 的每个执行步骤，最终以表格形式呈现查询结果。

## 功能

- 悬浮输入框，回车即可提问
- 实时展示 Agent 执行进度（运行中 / 成功 / 失败）
- 查询结果以表格渲染，支持横向滚动、表头吸顶
- 错误信息友好提示

## 开发

```bash
npm install
npm run dev
```

默认运行在 <http://localhost:5173>。

开发服务器已在 [`vite.config.js`](vite.config.js) 中配置代理，将 `/api` 转发到后端 `http://localhost:8001`，因此**需要先启动后端**。

## 构建

```bash
npm run build      # 产物输出到 dist/
npm run preview    # 本地预览构建产物
```

## 后端依赖

前端调用 `POST /api/query`，请求体为 `{ "query": "统计去年各地区的销售总额" }`，响应为 SSE 流式事件（`progress` / `result` / `error`）。

完整的启动步骤（中间件、知识库构建、后端）请参见[项目根目录 README](../README.md)。
