# 版本号接口与前端展示 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **执行方式（用户确认，2026-10-15）：** Subagent 驱动（subagent-driven-development）——每任务派发独立子代理执行，任务间做 review。

**Goal:** 后端（Go+Gin）暴露 `GET /api/version` 返回硬编码 semver 版本号，前端（Vue 3 + Vite）调用该接口并渲染展示。

**Architecture:** 两个独立仓：`version-backend` 是纯 API 服务（Gin，监听 `:8080`），`version` 是纯前端（Vue 3 + Vite）。两端仅通过单一接口契约 `GET /api/version` 耦合；Vite dev server 将 `/api` 代理到 `http://localhost:8080`。

**Tech Stack:** Go 1.21+、Gin、net/http/httptest；Vue 3、Vite 5、@vitejs/plugin-vue。

## Global Constraints

- 接口契约（唯一仓间对齐点）：`GET /api/version` → `200`，`Content-Type: application/json`，响应体 `{"version":"1.0.0"}`。
- 版本号来源：后端硬编码语义化 semver 常量，值固定为 `"1.0.0"`。
- 后端监听端口：`:8080`；前端 dev 代理 `/api` → `http://localhost:8080`。
- Git 写操作（commit/push/merge 等）在本次任务中被硬性禁止：本计划中所有「Commit」步骤跳过，仅落盘完成改动。
- 两仓改动分别落在各自 worktree：前端 `version-main/`，后端 `version-backend-main/`。

---

### Task 1: version-backend — Gin 版本号接口

**Files:**
- Create: `version-backend-main/go.mod`
- Create: `version-backend-main/main.go`
- Create: `version-backend-main/version.go`
- Test: `version-backend-main/version_test.go`

**Interfaces:**
- Consumes: 无（首个任务）。
- Produces:
  - 常量 `AppVersion string = "1.0.0"`（`version.go`）
  - 处理器 `versionHandler(c *gin.Context)`（`version.go`）
  - 中间件 `corsMiddleware() gin.HandlerFunc`（`main.go`）
  - HTTP 端点 `GET /api/version` → `{"version":"1.0.0"}`（后续 Task 2 前端依赖的契约）

- [ ] **Step 1: 初始化 Go module 并拉取依赖**

Run:
```bash
cd version-backend-main && go mod init version-backend && go get github.com/gin-gonic/gin@v1.9.1
```
Expected: 生成 `go.mod`，`require github.com/gin-gonic/gin v1.9.1` 写入并生成 `go.sum`。

- [ ] **Step 2: 编写失败的测试**

Create `version-backend-main/version_test.go`:

```go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/gin-gonic/gin"
)

func TestVersionEndpoint(t *testing.T) {
	gin.SetMode(gin.TestMode)
	r := gin.Default()
	r.GET("/api/version", versionHandler)

	req := httptest.NewRequest(http.MethodGet, "/api/version", nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("expected status 200, got %d", w.Code)
	}
	if ct := w.Header().Get("Content-Type"); ct != "application/json; charset=utf-8" {
		t.Fatalf("expected application/json content-type, got %q", ct)
	}
	want := `{"version":"1.0.0"}`
	if got := w.Body.String(); got != want {
		t.Fatalf("expected body %s, got %s", want, got)
	}
}
```

- [ ] **Step 3: 运行测试确认失败**

Run:
```bash
cd version-backend-main && go test ./... -v
```
Expected: FAIL，报 `undefined: versionHandler`。

- [ ] **Step 4: 编写最小实现**

Create `version-backend-main/version.go`:

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// AppVersion 是后端硬编码的语义化版本号（semver）。
const AppVersion = "1.0.0"

func versionHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{"version": AppVersion})
}
```

Create `version-backend-main/main.go`:

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.Use(corsMiddleware())
	r.GET("/api/version", versionHandler)
	r.Run(":8080")
}

func corsMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
		c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
		c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type")
		if c.Request.Method == "OPTIONS" {
			c.AbortWithStatus(204)
			return
		}
		c.Next()
	}
}
```

- [ ] **Step 5: 运行测试确认通过**

Run:
```bash
cd version-backend-main && go test ./... -v
```
Expected: PASS（`TestVersionEndpoint` 通过）。

- [ ] **Step 6: 编译冒烟检查**

Run:
```bash
cd version-backend-main && go build ./...
```
Expected: 编译成功，无输出错误。

（Commit 步骤因 Global Constraints「Git 只读」跳过。）

---

### Task 2: version — Vue 3 + Vite 前端调用并渲染

**Files:**
- Create: `version-main/package.json`
- Create: `version-main/vite.config.js`
- Create: `version-main/index.html`
- Create: `version-main/src/main.js`
- Create: `version-main/src/App.vue`

**Interfaces:**
- Consumes: Task 1 的接口契约 `GET /api/version` → `{"version":"1.0.0"}`。
- Produces: 前端页面加载后调用该接口、渲染版本号字符串；`npm run build` 可打包通过。

- [ ] **Step 1: 创建 package.json**

Create `version-main/package.json`:

```json
{
  "name": "version",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "vue": "^3.4.21"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.4",
    "vite": "^5.2.0"
  }
}
```

- [ ] **Step 2: 创建 vite.config.js（含 /api 代理）**

Create `version-main/vite.config.js`:

```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true
      }
    }
  }
})
```

- [ ] **Step 3: 创建 index.html**

Create `version-main/index.html`:

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>版本号展示</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

- [ ] **Step 4: 创建 src/main.js 与 src/App.vue**

Create `version-main/src/main.js`:

```js
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

Create `version-main/src/App.vue`:

```vue
<script setup>
import { ref, onMounted } from 'vue'

const version = ref('')
const loading = ref(true)
const error = ref('')

onMounted(async () => {
  try {
    const res = await fetch('/api/version')
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const data = await res.json()
    if (typeof data.version !== 'string' || data.version === '') {
      throw new Error('missing version field')
    }
    version.value = data.version
  } catch {
    error.value = '获取版本号失败'
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <main>
    <h1>版本号</h1>
    <p v-if="loading">加载中…</p>
    <p v-else-if="error">{{ error }}</p>
    <p v-else>{{ version }}</p>
  </main>
</template>
```

- [ ] **Step 5: 安装依赖并构建验证**

Run:
```bash
cd version-main && npm install && npm run build
```
Expected: 安装成功，`vite build` 输出 `dist/index.html` 等产物、无报错。

- [ ] **Step 6: 端到端冒烟（手动，开发环境）**

Run（两个终端）:
```bash
# 终端 1：后端
cd version-backend-main && go run .
# 终端 2：前端
cd version-main && npm run dev
```
然后访问前端 dev server 地址（默认 `http://localhost:5173`）。
Expected: 页面渲染出版本号 `1.0.0`（Vite 代理把 `/api/version` 转发到 `:8080`）。

（Commit 步骤因 Global Constraints「Git 只读」跳过。）