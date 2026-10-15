# 版本号接口与前端展示 — 设计说明书

- 日期：2026-10-15
- 涉及仓库：`version`（前端）、`version-backend`（后端）
- 需求：后端提供版本号接口，前端调用后端获取版本号并渲染展示。

## 1. 目标与范围

本设计落地一个最小可用的端到端链路：后端暴露版本号 HTTP 接口，前端在页面加载时调用该接口并把版本号渲染到页面。范围聚焦于单接口 + 单页面渲染，不含认证、多环境配置中心、版本号自动注入等超纲能力。

## 2. 架构概览

```
浏览器(Vue 3 + Vite) --GET /api/version--> Vite dev proxy --> Gin(:8080) --> 常量 AppVersion
                  <--200 {"version":"1.0.0"}--  JSON handler  <--
```

- 两仓独立：`version-backend` 是纯 API 服务，`version` 是纯前端。
- 耦合仅发生在单一接口契约上，两端可独立开发、测试与部署。

## 3. 接口契约（仓间对齐点）

- 方法/路径：`GET /api/version`
- 成功响应：`200`，`Content-Type: application/json`，响应体 `{"version":"1.0.0"}`
- 版本号来源：后端硬编码语义化 semver 常量
- 失败响应：服务内部异常时返回 `500`，响应体 `{"error":"internal error"}`

## 4. 后端 `[version-backend]`（Go + Gin）

### 组件

- `main.go`：初始化 Gin Engine，注册中间件与路由，监听 `:8080`。
- `version.go`（handler 层）：
  - 常量 `AppVersion = "1.0.0"`（semver，单点维护）。
  - 处理 `GET /api/version`，返回 JSON。
- CORS 中间件：放开跨域，便于前端（不同源）调试与部署访问。

### 数据流

请求进入 → Gin 路由匹配 `/api/version` → handler 读取 `AppVersion` → 写回 `{"version": "..."}`。

### 错误处理

handler 内部逻辑足够简单，唯一失败路径为 JSON 编码异常；统一按 500 返回。

### 测试

- 使用 `net/http/httptest` + `httptest.NewRecorder` 对真实 Router 发起 `GET /api/version`。
- 断言：状态码 200、`Content-Type` 为 JSON、响应体 `version == "1.0.0"`。

## 5. 前端 `[version]`（Vue 3 + Vite）

### 组件

- `App.vue`：唯一页面组件，承载版本号展示。
- `vite.config.js`：dev server 配置 `/api` 代理到 `http://localhost:8080`。

### 数据流

组件挂载 → `fetch('/api/version')` → 成功则解析 `data.version` 并渲染；失败进入错误态。

### 错误处理与状态

- 加载中：请求未返回时显示「加载中…」。
- 失败：网络错误 / 非 200 / 响应缺 `version` 字段时显示「获取版本号失败」。

### 测试/构建

- 以 `npm install` + `npm run build` 通过为验收基线（产物可正常打包）。

## 6. 验收标准

1. 后端：`GET /api/version` 返回 `200` 且响应体为 `{"version":"1.0.0"}`。
2. 前端：构建通过；运行时能请求该接口并渲染版本号 `1.0.0`。
3. 两仓均在各自 worktree 内改动，接口契约保持一致。

## 7. 明确不做（YAGNI）

- 不接入认证/鉴权。
- 不引入版本号自动注入（Git tag / 构建元数据）。
- 不做多接口、多页面或状态管理库。