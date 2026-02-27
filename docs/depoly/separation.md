---
sidebar_position: 5
---

# 前后端分离

CBCTF 默认将前端静态资源嵌入 Go 二进制，通过同一服务端口提供服务。如需将前端托管至 CDN 或 GitHub Pages 等静态托管服务，可以采用前后端分离部署方式。

## 编译前端

CBCTF 前端代码位于主仓库的 `frontend/` 子目录。

```bash
git clone https://github.com/0RAYS/CBCTF.git
cd CBCTF/frontend
pnpm install
```

编辑 `src/api/config.js`，将 `BASE_URL` 指向后端服务地址，**不含末尾 `/`**：

```javascript
export const API_CONFIG = {
  // 后端 API 地址
  BASE_URL: 'https://api.ctf.example.com',
  TIMEOUT: 10000,
  RESPONSE_CODE: {
    SUCCESS: 200,
    UNAUTHORIZED: 401,
    FORBIDDEN: 403,
    NOT_FOUND: 404,
    SERVER_ERROR: 500,
  },
  DEFAULT_HEADERS: {
    'Content-Type': 'application/json',
  },
};
```

编译：

```bash
pnpm run build
```

构建产物位于 `dist/` 目录，将其部署至任意静态托管服务。

## 后端配置

编辑 `config.yaml`，将 `host` 指向后端地址，并将前端域名添加至 CORS 白名单：

```yaml
host: https://api.ctf.example.com

gin:
  cors:
    - https://ctf.example.com   # 前端部署地址
```

:::info
分离部署时，后端 `host` 填写 API 服务域名；前端通过 `BASE_URL` 与后端通信，后端通过 `gin.cors` 允许来自前端域名的跨域请求。
:::

## 使用 Helm 时的分离部署

若通过 Helm 部署后端，在 `values.yaml` 中同样配置：

```yaml
cbctf:
  host: "https://api.ctf.example.com"
  gin:
    cors:
      - "https://ctf.example.com"
```
