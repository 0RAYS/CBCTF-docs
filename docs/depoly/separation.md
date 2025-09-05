---
sidebar_position: 3
---

# 前后端分离

CBCTF 可将前端托管至 `Github Pages` 等静态服务，一定程度上减轻对服务器静态资源请求的压力

## 准备前端

下载前端代码，并拉取依赖

```bash
git clone https://github.com/0RAYS/CBCTF-frontend.git
pnpm install
```

编辑 `src/api/config.js`，**注意结尾不包含 `/`**

```javascript
export const API_CONFIG = {
  // API基础URL
  BASE_URL: 'https://your.backend.host',
  // 超时时间
  TIMEOUT: 10000,
  // 响应码
  RESPONSE_CODE: {
    SUCCESS: 200,
    UNAUTHORIZED: 401,
    FORBIDDEN: 403,
    NOT_FOUND: 404,
    SERVER_ERROR: 500,
  },
  // 默认headers
  DEFAULT_HEADERS: {
    'Content-Type': 'application/json',
  },
};
```

编译前端代码

```bash
pnpm run build
```

此时，静态前端资源将被存放在 `dist` 目录下

## 后端配置

编辑 `config.yaml`

```yaml
backend: https://your.backend.host
frontend: https://your.frontend.host
```
