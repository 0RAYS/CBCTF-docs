---
sidebar_position: 2
---

# 快速上手

CBCTF 提供三种部署方式：

| 方式 | 适用场景 |
|------|---------|
| **Docker** | 本地快速体验、小规模部署 |
| **Helm（推荐）** | 生产级 Kubernetes 部署，自动管理 MySQL/Redis |
| **二进制** | 已有 K8s 集群、需要自定义部署的场景 |

---

## Docker 部署

### 前置条件

- Docker 24+
- Docker Compose v2
- Kubernetes 集群（用于动态容器题目）

### 步骤

1. 创建 `docker-compose.yaml`：

   ```yaml
   services:
     cbctf:
       image: ghcr.io/0rays/cbctf:latest
       ports:
         - "8000:8000"
       volumes:
         - ./config.yaml:/app/config.yaml
         - ./data:/app/data
         - ./admin.yaml:/app/admin.yaml   # K8s kubeconfig
       depends_on:
         mysql:
           condition: service_healthy
         redis:
           condition: service_started

     mysql:
       image: mysql:8.0
       environment:
         MYSQL_ROOT_PASSWORD: rootpassword
         MYSQL_DATABASE: cbctf
         MYSQL_USER: cbctf
         MYSQL_PASSWORD: cbctf_password
       volumes:
         - mysql_data:/var/lib/mysql
       healthcheck:
         test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
         interval: 10s
         timeout: 5s
         retries: 5

     redis:
       image: redis:7-alpine
       command: redis-server --requirepass redis_password
       volumes:
         - redis_data:/data

   volumes:
     mysql_data:
     redis_data:
   ```

2. 创建 `config.yaml`（完整字段说明见 [配置说明](/docs/depoly/settings)）：

   ```yaml
   host: http://your.domain.com

   path: /app/data

   gin:
     host: 0.0.0.0
     port: 8000
     cors:
       - http://your.domain.com
     jwt:
       secret: change-this-to-a-random-string

   gorm:
     mysql:
       host: mysql
       port: 3306
       user: cbctf
       pwd: cbctf_password
       db: cbctf

   redis:
     host: redis
     port: 6379
     pwd: redis_password

   k8s:
     config: /app/admin.yaml
     namespace: cbctf
   ```

3. 将 K8s 集群的 kubeconfig 复制为 `./admin.yaml`。

4. 启动服务：

   ```bash
   docker compose up -d
   ```

5. 查看管理员初始密码：

   ```bash
   docker compose logs cbctf | grep "Init Admin"
   ```

---

## Helm 部署（推荐）

适用于生产 Kubernetes 环境，Helm Chart 会自动部署 MySQL、Redis 并完成配置注入。

详见 [Helm 部署](/docs/depoly/helm)。

---

## 二进制部署

适用于已有 K8s 集群、但不使用 Helm 的场景，需自行准备 MySQL 和 Redis。

### 前置条件

- Go 1.26+
- Node.js 22+，pnpm v10+
- MySQL 8.0+
- Redis 6.0+
- Kubernetes 集群（参考 [集群部署](/docs/depoly/depoly.md)）

### 编译

1. 克隆仓库：

   ```bash
   git clone https://github.com/0RAYS/CBCTF.git
   cd CBCTF
   ```

2. 编译前端（构建产物将被 Go 二进制嵌入）：

   ```bash
   cd frontend
   pnpm install
   pnpm run build
   cd ..
   ```

3. 编译后端：

   ```bash
   # 推荐：含流量捕获功能（需要 libpcap-dev）
   # Ubuntu: sudo apt install -y libpcap-dev
   CGO_ENABLED=1 go build \
     -ldflags="-s -w -linkmode external -extldflags '-static'" \
     -trimpath -o cbctf .

   # 不含流量捕获（无需 CGO）
   go build -ldflags="-s -w" -trimpath -o cbctf .
   ```

### 配置

首次运行会自动生成 `config.yaml`，根据实际环境修改后重新运行。完整字段说明见 [配置说明](/docs/depoly/settings)。

```yaml
host: https://your.domain.com    # 平台对外访问地址（OAuth 回调、邮件链接等均使用此地址）

path: ./data                      # 数据存储目录，需具备读写权限

gin:
  host: 0.0.0.0
  port: 8000
  proxies:
    - 10.0.0.1                    # 反向代理服务器 IP，用于获取真实客户端 IP
  cors:
    - https://your.domain.com
  jwt:
    secret: change-this-to-a-random-string

gorm:
  mysql:
    host: 127.0.0.1
    port: 3306
    user: cbctf
    pwd: your-mysql-password
    db: cbctf

redis:
  host: 127.0.0.1
  port: 6379
  pwd: your-redis-password

k8s:
  config: ./admin.yaml            # K8s 集群 kubeconfig 路径
  namespace: cbctf
```

### 运行

```bash
# 默认加载 ./config.yaml
./cbctf

# 指定配置文件路径
./cbctf -c /path/to/config.yaml
```

### 后台运行（systemd）

创建 `/etc/systemd/system/cbctf.service`：

```ini
[Unit]
Description=CBCTF Service
After=network.target

[Service]
Type=simple
User=cbctf
WorkingDirectory=/opt/cbctf
ExecStart=/opt/cbctf/cbctf
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cbctf
```

---

## 初始化管理员

无论使用哪种部署方式，当数据库中不存在管理员账号时，服务启动后将自动创建管理员并将凭据打印至日志：

```
Init Admin: Admin{ name: admin, password: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx, email: admin@0rays.club}
```

管理员登录地址：`https://your.domain.com/platform/#/admin/login`

:::warning
请在首次登录后立即修改初始密码。
:::
