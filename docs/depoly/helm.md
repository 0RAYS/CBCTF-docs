---
sidebar_position: 4
---

# Helm 部署

CBCTF 提供官方 Helm Chart，可一键在 Kubernetes 集群上部署完整的 CBCTF 平台，包含 MySQL、Redis 及所有必要的 RBAC 配置。

## 前置条件

- Kubernetes 集群（参考 [集群部署](/docs/depoly/depoly.md)）
- Helm v3.10+
- 集群已安装：
  - Multus CNI（VPC 网络模式必需）
  - Kube-OVN（VPC 网络模式必需）
  - 支持 `ReadWriteMany` 的 StorageClass（NFS 等）

## 安装

### 添加 Chart 仓库

```bash
helm repo add cbctf https://0rays.github.io/CBCTF-charts
helm repo update
```

### 快速安装

```bash
helm install cbctf cbctf/cbctf \
  --namespace cbctf \
  --create-namespace \
  --set cbctf.host=https://ctf.example.com \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=ctf.example.com
```

### 使用自定义 values 安装（推荐）

```bash
# 下载默认 values 文件
helm show values cbctf/cbctf > my-values.yaml

# 编辑 my-values.yaml 后安装
helm install cbctf cbctf/cbctf \
  --namespace cbctf \
  --create-namespace \
  -f my-values.yaml
```

### 查看初始管理员密码

```bash
kubectl logs -n cbctf deploy/cbctf | grep "Init Admin"
```

---

## 升级

```bash
helm upgrade cbctf cbctf/cbctf \
  --namespace cbctf \
  -f my-values.yaml
```

## 卸载

```bash
helm uninstall cbctf --namespace cbctf
```

:::caution
卸载不会自动删除 PVC（持久化存储），如需清理数据需手动删除：
```bash
kubectl delete pvc --all -n cbctf
```
:::

---

## Values 完整说明

### 镜像配置

```yaml
image:
  repository: ghcr.io/0rays/cbctf
  pullPolicy: IfNotPresent
  tag: "latest"
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `image.repository` | `ghcr.io/0rays/cbctf` | 镜像仓库地址 |
| `image.pullPolicy` | `IfNotPresent` | 镜像拉取策略 |
| `image.tag` | `latest` | 镜像标签，建议固定版本（如 `20250101`） |

### 私有镜像仓库

若镜像存储在私有仓库，配置 `imageCredentials`，Chart 会自动创建 `docker-registry` 类型的 Secret：

```yaml
imageCredentials:
  registry: ghcr.io
  username: your-username
  password: your-token
```

### 时区

```yaml
timezone: "Asia/Shanghai"   # 留空则使用容器默认时区（UTC）
```

### 网络服务

```yaml
service:
  type: ClusterIP   # ClusterIP / NodePort / LoadBalancer
  port: 8000

ingress:
  enabled: false
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: ctf.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: cbctf-tls
      hosts:
        - ctf.example.com
```

### CBCTF 应用配置

以下所有 `cbctf.*` 字段最终会被注入为 `config.yaml`，字段语义与 [配置说明](/docs/depoly/settings) 完全对应。

#### 基础配置

```yaml
cbctf:
  host: "https://ctf.example.com"   # 平台对外访问地址，必填

  log:
    level: info      # DEBUG / INFO / WARNING / ERROR
    save: true
```

#### Gin HTTP 服务

```yaml
cbctf:
  gin:
    mode: release    # debug / release / test
    host: "0.0.0.0"
    port: 8000
    upload:
      max: 8         # 单文件上传限制（MiB）
    proxies:
      - "127.0.0.1"  # 反向代理 IP，通常填 Ingress Controller 的 Pod CIDR
    cors:
      - "https://ctf.example.com"
    ratelimit:
      global: 100    # 全局限速（req/s）
      whitelist:
        - "::1"
        - "127.0.0.1"
    log:
      whitelist:
        - "/metrics"
        - "/platform/*filepath"
    jwt:
      secret: ""     # 留空则 Helm 自动生成随机密钥
      static: false
    metrics:
      whitelist:
        - "127.0.0.1"
        - "10.0.0.0/8"
        - "172.16.0.0/12"
        - "192.168.0.0/16"
```

:::info JWT 密钥
`cbctf.gin.jwt.secret` 留空时，Helm 会自动生成随机密钥并存储在 Kubernetes Secret 中（`helm upgrade` 时保持不变）。生产环境建议显式设置。
:::

#### 异步任务队列

```yaml
cbctf:
  asynq:
    concurrency: 50
    log:
      level: warning
```

#### 数据库日志

```yaml
cbctf:
  gorm:
    log:
      level: silent   # INFO / WARNING / ERROR / SILENT
```

MySQL 连接信息由 Helm 从 `mysql.auth.*` 自动注入，无需手动配置。

#### Kubernetes 集成

```yaml
cbctf:
  k8s:
    tcpdump: "nicolaka/netshoot:latest"
    generator_worker: 2

    # FRP 内网穿透（可选）
    frp:
      on: false
      frpc: "snowdreamtech/frpc:latest"
      nginx: "nginx:latest"
      frps:
        - host: frps.example.com
          port: 7000
          token: your-token
          allowed:
            - from: 10000
              to: 30000
              exclude:
                - 20000

    # 启用 KubeOVN RBAC（使用 VPC 网络模式时必须设为 true）
    kubeovnRBAC: false

    # 启用 Multus RBAC（使用 VPC 网络模式时必须设为 true）
    multusRBAC: false

    # 外部网络配置（VPC 模式中允许靶机访问外部网络所需）
    externalNetwork:
      enabled: false
      cidr: "192.168.0.0/16"    # 宿主机所在网段
      gateway: "192.168.0.1"    # 宿主机网关
      interface: "eth0"          # 宿主机网卡名称（所有节点须一致）
```

:::info VPC 网络模式
若需使用 VPC 网络隔离靶机，必须同时设置：
- `cbctf.k8s.kubeovnRBAC: true`
- `cbctf.k8s.multusRBAC: true`
- `cbctf.k8s.externalNetwork.enabled: true` 并填写正确的网络参数
:::

#### 作弊检测

```yaml
cbctf:
  cheat:
    ip:
      whitelist:
        - "127.0.0.1"
        - "::1"
        - "10.0.0.0/8"
        - "192.168.0.0/16"
        - "172.16.0.0/12"
        - "100.64.0.0/10"
```

#### 注册配置

```yaml
cbctf:
  registration:
    enabled: true       # false 则禁止公开注册
    default_group: 0    # 新用户默认分组 ID
```

#### Webhook

```yaml
cbctf:
  webhook:
    whitelist: []   # 空列表表示不限制目标地址
```

### 持久化存储

CBCTF 需要 `ReadWriteMany` 类型的 PVC 用于多 Pod 间共享附件文件（当使用多副本或动态附件生成时）。

```yaml
persistence:
  enabled: true
  storageClass: ""        # 留空使用集群默认 StorageClass
  accessMode: ReadWriteMany
  size: 20Gi
  # existingClaim: ""    # 使用已有 PVC 时取消注释
```

:::warning StorageClass 要求
K3S 默认的 `local-path` StorageClass **不支持** `ReadWriteMany`。部署前需配置支持 RWX 的 StorageClass（如 NFS Provisioner）并将其设为默认，或在 `persistence.storageClass` 中显式指定。
:::

### MySQL

Chart 内置 MySQL StatefulSet，可通过 `mysql.enabled: false` 禁用并使用外部 MySQL。

```yaml
mysql:
  enabled: true
  image:
    repository: mysql
    tag: "8.0"
    pullPolicy: IfNotPresent
  auth:
    rootPassword: ""    # 留空自动生成
    database: "cbctf"
    username: "cbctf"
    password: ""        # 留空自动生成
  persistence:
    enabled: true
    storageClass: ""
    accessMode: ReadWriteOnce
    size: 5Gi
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 512Mi
  service:
    port: 3306
  extraConfig: |
    [mysqld]
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
    max_connections = 500
    innodb_buffer_pool_size = 256M
```

使用外部 MySQL 时（`mysql.enabled: false`），需通过环境变量手动注入连接信息：

```yaml
mysql:
  enabled: false

# 在 cbctf 的 extraEnvs 中注入（或直接修改 ConfigMap 后重启）
# 环境变量优先级高于 config.yaml
# CBCTF_GORM_MYSQL_HOST / _PORT / _USER / _PWD / _DB
```

### Redis

```yaml
redis:
  enabled: true
  image:
    repository: redis
    tag: "7-alpine"
    pullPolicy: IfNotPresent
  auth:
    password: ""    # 留空自动生成
  persistence:
    enabled: true
    storageClass: ""
    accessMode: ReadWriteOnce
    size: 1Gi
  resources: {}
  service:
    port: 6379
```

### 资源限制

```yaml
resources:
  limits:
    cpu: 2000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 256Mi
```

### 其他 Pod 配置

```yaml
nodeSelector: {}
tolerations: []
affinity: {}
podAnnotations: {}
podLabels: {}
```

---

## 常见场景示例

### 最小化部署（ClusterIP + port-forward）

```yaml
cbctf:
  host: "http://localhost:8000"
  gin:
    cors:
      - "http://localhost:8000"
    jwt:
      secret: "my-secret"

mysql:
  auth:
    password: "mysql-password"

redis:
  auth:
    password: "redis-password"
```

```bash
helm install cbctf cbctf/cbctf -f values.yaml -n cbctf --create-namespace
kubectl port-forward svc/cbctf 8000:8000 -n cbctf
```

### 生产部署（Ingress + TLS + VPC 网络）

```yaml
cbctf:
  host: "https://ctf.example.com"
  gin:
    cors:
      - "https://ctf.example.com"
    jwt:
      secret: "your-very-long-random-secret"
    proxies:
      - "10.244.0.0/16"    # Ingress Controller Pod CIDR
  k8s:
    kubeovnRBAC: true
    multusRBAC: true
    externalNetwork:
      enabled: true
      cidr: "192.168.0.0/24"
      gateway: "192.168.0.1"
      interface: "eth0"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: ctf.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: cbctf-tls
      hosts:
        - ctf.example.com

persistence:
  storageClass: "nfs-client"
  size: 50Gi

mysql:
  auth:
    rootPassword: "strong-root-password"
    password: "strong-cbctf-password"
  persistence:
    size: 20Gi

redis:
  auth:
    password: "strong-redis-password"
```
