---
sidebar_position: 2
---

# 配置说明

CBCTF 的核心配置文件为程序根目录下的 `config.yaml`，使用 YAML 格式。

所有配置项均可通过以 `CBCTF_` 为前缀的环境变量覆盖，使用 `_` 作为层级分隔符。例如：
- `CBCTF_GIN_PORT=9000` 覆盖 `gin.port`
- `CBCTF_GORM_MYSQL_PWD=newpassword` 覆盖 `gorm.mysql.pwd`

## 完整默认配置

```yaml
host: http://127.0.0.1:8000

path: ./data

log:
  level: info
  save: true

asynq:
  concurrency: 50
  log:
    level: warning

gin:
  mode: release
  host: 127.0.0.1
  port: 8000
  proxies:
    - 127.0.0.1
  upload:
    max: 8
  ratelimit:
    global: 100
    whitelist:
      - "::1"
      - 127.0.0.1
  cors:
    - http://127.0.0.1:8000
  log:
    whitelist:
      - /metrics
      - /platform/*filepath
  jwt:
    secret: change-this-secret
    static: false
  metrics:
    whitelist:
      - 127.0.0.1
      - "::1"

gorm:
  mysql:
    host: 127.0.0.1
    port: 3306
    user: cbctf
    pwd: password
    db: cbctf
    mxopen: 100
    mxidle: 10
  log:
    level: silent

redis:
  host: 127.0.0.1
  port: 6379
  pwd: password

k8s:
  config: ./admin.yaml
  namespace: cbctf
  tcpdump: nicolaka/netshoot:latest
  generator_worker: 2
  frp:
    on: false
    frpc: snowdreamtech/frpc:latest
    nginx: nginx:latest
    frps: []

cheat:
  ip:
    whitelist:
      - 127.0.0.1
      - "::1"
      - 10.0.0.0/8
      - 192.168.0.0/16
      - 172.16.0.0/12
      - 100.64.0.0/10

webhook:
  whitelist: []

registration:
  enabled: true
  default_group: 0

geocity_db: ./data/GeoLite2-City.mmdb
```

---

## 字段说明

### `host`

平台对外公开访问地址，用于 OAuth 回调、邮件链接、前端跨域等场景。**不含末尾斜线。**

```yaml
host: https://ctf.example.com
```

---

### `path`

程序数据存储根目录，程序需要对该目录具有完整读写权限。存储内容包括：上传的附件、日志（当 `log.save: true` 时）、GeoIP 数据库等。

```yaml
path: ./data
```

---

### `log`

全局日志配置。

```yaml
log:
  level: info    # 终端日志等级：DEBUG / INFO / WARNING / ERROR
  save: true     # 是否将日志持久化至 ./logs 目录
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `level` | string | `info` | 日志等级，支持 `DEBUG` `INFO` `WARNING` `ERROR` |
| `save` | bool | `true` | 是否保存日志文件至 `./logs/` |

---

### `asynq`

异步任务队列配置（基于 Redis 的 [Asynq](https://github.com/hibiken/asynq) 框架）。

```yaml
asynq:
  concurrency: 50   # 最大并发 worker 数
  log:
    level: warning  # Asynq 自身日志等级：DEBUG / INFO / WARNING / ERROR
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `concurrency` | int | `50` | 最大并发任务数，过低会导致任务堆积，过高会导致服务器过载 |
| `log.level` | string | `warning` | Asynq 内部日志等级 |

---

### `gin`

HTTP 服务器配置。

```yaml
gin:
  mode: release
  host: 0.0.0.0
  port: 8000
  proxies:
    - 10.0.0.1
  upload:
    max: 8
  ratelimit:
    global: 100
    whitelist:
      - "::1"
      - 127.0.0.1
  cors:
    - https://ctf.example.com
  log:
    whitelist:
      - /metrics
      - /platform/*filepath
  jwt:
    secret: your-secret-key
    static: false
  metrics:
    whitelist:
      - 127.0.0.1
      - 10.0.0.0/8
```

#### `gin.mode`

Gin 运行模式，影响框架日志输出。

| 值 | 说明 |
|----|------|
| `release` | 生产模式（默认，推荐） |
| `debug` | 调试模式，输出详细路由日志 |
| `test` | 测试模式 |

#### `gin.host`

HTTP 服务监听地址。生产环境通常设为 `0.0.0.0` 监听所有接口；若有反向代理则可仅监听 `127.0.0.1`。

#### `gin.port`

HTTP 服务监听端口，默认 `8000`。

#### `gin.proxies`

可信反向代理服务器 IP 列表，支持 CIDR 格式。配置后，平台才能从 `X-Forwarded-For` 等头部正确获取真实客户端 IP（用于作弊检测、IP 白名单等）。

```yaml
gin:
  proxies:
    - 10.0.0.1         # 反向代理 IP
    - 172.16.0.0/12    # 或 CIDR 段
```

#### `gin.upload.max`

单次文件上传大小限制，单位 MiB，默认 `8`。

#### `gin.ratelimit`

请求速率限制。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `global` | int | `100` | 全局限速（req/s），超出返回 `429`；部分接口有独立限制 |
| `whitelist` | []string | `["::1", "127.0.0.1"]` | 不受限速约束的 IP 列表，支持 CIDR |

#### `gin.cors`

允许跨域访问的前端地址列表。当前后端分离部署时，需将前端域名加入此列表。

```yaml
gin:
  cors:
    - https://ctf.example.com
    - https://www.ctf.example.com
```

#### `gin.log.whitelist`

不记录访问日志的路径列表。默认排除 `/metrics` 和静态资源路径，避免日志噪声。路径须与[路由定义](https://github.com/0RAYS/CBCTF/blob/main/internal/router/router.go)完全匹配，支持 `*` 通配。

#### `gin.jwt`

JWT 认证配置。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `secret` | string | — | JWT 签名密钥，**务必修改为随机字符串**，泄露将导致认证绕过 |
| `static` | bool | `false` | 为 `true` 时禁止动态更新 JWT 密钥（不推荐） |

#### `gin.metrics`

Prometheus 指标接口访问控制。

```yaml
gin:
  metrics:
    whitelist:
      - 127.0.0.1
      - 10.0.0.0/8    # Prometheus 所在网段
```

只有白名单内的 IP 才能访问 `/metrics` 端点。

---

### `gorm`

数据库（MySQL）配置。

```yaml
gorm:
  mysql:
    host: 127.0.0.1
    port: 3306
    user: cbctf
    pwd: password
    db: cbctf
    mxopen: 100   # 最大连接数
    mxidle: 10    # 最大空闲连接数
  log:
    level: silent
```

#### `gorm.mysql`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `host` | string | `127.0.0.1` | MySQL 主机地址 |
| `port` | int | `3306` | MySQL 端口 |
| `user` | string | `cbctf` | 数据库用户名 |
| `pwd` | string | `password` | 数据库密码 |
| `db` | string | `cbctf` | 数据库名 |
| `mxopen` | int | `100` | 连接池最大连接数 |
| `mxidle` | int | `10` | 连接池最大空闲连接数 |

#### `gorm.log.level`

GORM 日志等级，支持 `INFO` `WARNING` `ERROR` `SILENT`，默认 `SILENT`。

---

### `redis`

Redis 配置。

```yaml
redis:
  host: 127.0.0.1
  port: 6379
  pwd: password
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `host` | string | `127.0.0.1` | Redis 主机地址 |
| `port` | int | `6379` | Redis 端口 |
| `pwd` | string | — | Redis 密码，无密码时留空 |

---

### `k8s`

Kubernetes 集成配置，用于动态容器题目的生命周期管理。

```yaml
k8s:
  config: ./admin.yaml         # kubeconfig 文件路径
  namespace: cbctf             # 部署题目容器的命名空间
  tcpdump: nicolaka/netshoot:latest  # 流量捕获辅助镜像
  generator_worker: 2          # 每个动态附件题目的生成器 Pod 数量倍率

  frp:
    on: false
    frpc: snowdreamtech/frpc:latest
    nginx: nginx:latest
    frps:
      - host: frps.example.com
        port: 7000
        token: your-token
        allowed:
          - from: 10000
            to: 30000
            exclude:
              - 20000
```

#### `k8s.config`

kubeconfig 文件路径，用于平台访问 Kubernetes API。使用 Helm 部署时，此路径由 init container 自动生成，无需手动配置。

#### `k8s.namespace`

动态题目容器部署的 Kubernetes 命名空间，同时作为相关 K8s 资源名称前缀，默认 `cbctf`。

#### `k8s.tcpdump`

流量捕获功能使用的辅助容器镜像，默认 `nicolaka/netshoot:latest`。在网络受限环境中可替换为私有镜像仓库地址。

#### `k8s.generator_worker`

动态附件生成器的 Pod 数量倍率。单个题目实际运行的 Pod 数 = `节点数 × generator_worker`，默认 `2`，需根据集群规模调整。

#### `k8s.frp`

FRP 内网穿透配置，用于将题目容器端口通过 frps 暴露给选手访问。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `on` | bool | `false` | 是否启用 FRP 功能 |
| `frpc` | string | `snowdreamtech/frpc:latest` | frpc 客户端容器镜像 |
| `nginx` | string | `nginx:latest` | Nginx 反向代理容器镜像（用于 HTTP 类题目） |
| `frps` | []object | `[]` | frps 服务器列表，多个时随机选择 |

`frps` 列表项字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `host` | string | frps 服务器地址 |
| `port` | int | frps 服务器端口 |
| `token` | string | frps 认证 Token |
| `allowed` | []object | 允许映射的端口范围列表 |
| `allowed[].from` | int | 端口范围起始 |
| `allowed[].to` | int | 端口范围结束 |
| `allowed[].exclude` | []int | 排除的具体端口 |

---

### `cheat`

作弊检测配置。

```yaml
cheat:
  ip:
    whitelist:
      - 127.0.0.1
      - "::1"
      - 10.0.0.0/8
      - 192.168.0.0/16
      - 172.16.0.0/12
      - 100.64.0.0/10
```

#### `cheat.ip.whitelist`

IP 白名单，白名单内的 IP 不参与 flag 共享检测。支持 CIDR 格式。默认包含所有私有地址段，防止内网 IP 误报。

---

### `webhook`

出站 Webhook 配置。

```yaml
webhook:
  whitelist: []    # 允许的 Webhook 目标地址（URL 前缀），空列表表示不限制
```

`whitelist` 为空列表时，允许向任意地址发送 Webhook；配置后仅允许向指定前缀的 URL 发送请求。

---

### `registration`

用户注册配置。

```yaml
registration:
  enabled: true      # 是否开放公开注册
  default_group: 0   # 新用户默认分组 ID（0 表示无分组）
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | `true` | 关闭后禁止新用户自行注册（管理员仍可手动创建） |
| `default_group` | int | `0` | 新注册用户自动加入的分组 ID，`0` 表示不分配分组 |

---

### `geocity_db`

GeoIP 城市数据库文件路径，用于对用户 IP 进行地理位置解析（事件日志、作弊检测等功能）。

```yaml
geocity_db: ./data/GeoLite2-City.mmdb
```

数据库文件需自行从 [MaxMind](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data) 下载后放置于 `path` 目录下。此字段非必填，缺少数据库时地理位置解析功能将不可用，其他功能不受影响。
