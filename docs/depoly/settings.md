---
sidebar_position: 2
---

# config.yml

为 CBCTF 的核心配置文件，位于程序根目录下 `config.yml`，使用 YAML 格式编写

## 默认配置

```yaml
backend: http://127.0.0.1:8000
# CORS
frontend: http://127.0.0.1:3000
# Data path and NFS mount path if needed
path: ./data
log:
  # DEBUG INFO WARNING ERROR
  level: info
  save: true
asynq:
  # DEBUG INFO WARNING ERROR
  level: warning
  # Concurrent workers
  concurrency: 50
gin:
  mode: release
  host: 127.0.0.1
  port: 8000
  # Trusted proxy servers
  proxies:
    - 10.0.0.1
  upload:
    # Size limited
    max: 8
  rate:
    # Global rate limit
    global: 100
    # Request without rate limited
    whitelist:
      - ::1
      - 127.0.0.1
  log:
    # Request without logging
    whitelist:
      - /metrics
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
    # INFO WARNING ERROR SILENT
    level: silent
redis:
  host: 127.0.0.1
  port: 6379
  pwd: password
k8s:
  # Kubeconfig path
  config: ./admin.conf
  # Also as prefix of resources
  namespace: cbctf
  # Maybe your k8s nodes cidr
  # https://kubeovn.github.io/docs/stable/vpc/vpc/#_2
  external_network:
    cidr: 192.168.0.0/16
    gateway: 192.168.0.1
    interface: eth0
    exclude_ips:
      - 192.168.0.1
      - 192.168.0.254
  # Docker image
  tcpdump: nicolaka/netshoot:latest
  frpc:
    on: false
    # Docker image
    frpc_image: snowdreamtech/frpc:latest
    nginx_image: nginx:latest
    frps:
      - host: example.com
        port: 7000
        token: token
        allowed_ports:
          - from: 10000
            to: 30000
            exclude:
              - 20000
          - from: 40000
            to: 60000
            exclude:
              - 50000
  # (len(nodes) * generator_worker) pods for each dynamic challenge
  generator_worker: 2
nfs:
  server: 127.0.0.1
  # Accessible path (RW) in NFS server
  path: /mnt/data
  # Gi, Mi
  storage: 10Gi
```

## 字段说明

### backend

后端服务地址，作为外部访问地址，也是 `Oauth` 重定向地址。例：

```text
https://training.0rays.club
```

### frontend

前端服务地址，作为外部访问地址，通常情况下与 `backend` 字段相同，仅当前后端分离部署时不同；同时作为 CORS 访问策略。例：

```text
https://training.0rays.club
```

### path

数据存储路径，作为程序数据存储位置，同时作为 `NFS` 挂载路径，程序需要有对其的完全访问控制权限。例：

```text
./data
```

### log

全局日志配置

#### level

终端日志等级，支持 `DEBUG` `INFO` `WARNING` `ERROR` 四种等级，默认 `INFO`

#### save

是否保存日志至文件，默认 `true`，日志文件位于 `./logs` 目录下

### asynq

异步任务队列配置

#### level

日志输出等级，支持 `DEBUG` `INFO` `WARNING` `ERROR` 四种等级，默认 `WARNING`

#### concurrency

最大并发任务数，默认 `50`，需根据服务器性能进行调整，过低会导致任务堆积，过高会导致服务器过载

该值并非主程最大 Goroutine 数，实际 Goroutine 数可能会大于该值

### gin

#### mode

运行模式，支持 `debug` `release` `test` 三种模式，默认 `release`，影响日志输出等级

#### host

监听地址

#### port

监听端口

#### proxies

可信代理服务器列表，支持 CIDR 格式，默认 `[]`，若部署在反向代理后面，需配置该字段，否则无法获取真实客户端 IP 地址

#### upload

文件上传配置

##### max

单次上传文件大小限制，单位 `MiB`，默认 `8`

#### rate

请求速率限制配置

##### global

全局请求速率限制，单位 `req/s`，默认 `100`，超过该值将返回 `429` 错误

部分接口有独立接口限制

##### whitelist

请求白名单，支持 CIDR 格式，默认 `["::1", "127.0.0.1"]`，白名单内的请求不受速率限制

#### log

请求日志配置

##### whitelist

请求日志白名单，默认 `["/metrics"]`，白名单内的请求不记录日志，主要用于排除监控请求

不支持前缀匹配，须为完整路径，对部分携带 URI 的路径须与[源代码](https://github.com/0RAYS/CBCTF/blob/main/internal/router/router.go)匹配

### gorm

数据库配置

#### mysql

MySQL 数据库配置

##### host

数据库地址

##### port

数据库端口

##### user

数据库用户名

##### pwd

数据库密码

##### db

数据库名称

##### mxopen

数据库最大连接数，默认 `100`，需根据服务器性能进行调整

##### mxidle

数据库最大空闲连接数，默认 `10`，需根据服务器性能进行调整

#### log

Gorm 日志配置

#### level

日志输出等级，支持 `INFO` `WARNING` `ERROR` `SILENT` 四种等级，默认 `SILENT`

### redis

Redis 配置

#### host

Redis 地址

#### port

Redis 端口

#### pwd

Redis 密码

### k8s

Kubernetes 配置

#### config

Kubeconfig 文件路径

#### namespace

Kubernetes 命名空间，同时作为部分资源名称前缀，默认 `cbctf`

#### tcpdump

用于网络抓包的 Docker 镜像，默认 `nicolaka/netshoot:latest`，可自行镜像

#### external_network

外部网络配置，允许靶机访问的外部网络，通常与宿主机所在网络相同，宿主机网络环境例：

```text
节点 IP 地址为: 10.233.8.1，网关地址为 10.233.0.1，网卡名称为 eth0
```

##### cidr

宿主机所在网段，例：10.233.0.0、16

##### gateway

宿主机网关地址，例：10.233.0.1

##### interface

宿主机网卡名称，例：eth0

##### exclude_ips

不会分配给靶机的 IP 地址，避免与宿主机 IP 地址冲突，例：`["10.233.8.1", "10.233.0.1"]`

#### frpc

FRP 相关配置

##### on

是否启用 FRP 功能，默认 `false`，启用后可通过 FRP 进行靶机端口映射

##### frpc_image

FRP 客户端 Docker 镜像，默认 `snowdreamtech/frpc:latest`，可自行镜像

##### nginx_image

Nginx 反向代理 Docker 镜像，默认 `nginx:latest`，可自行镜像

##### frps

FRP 服务器，可配置多个，随机选择一个进行内网穿透

###### host

FRP 服务器地址

###### port

FRP 服务器端口

###### token

FRP 服务器认证 Token

###### allowed_ports

FRP 服务器允许映射的端口范围

#### generator_worker

动态附件生成器容器数量倍率，单个题目实际运行的 Pod 数量为 `len(nodes) * generator_worker`，默认 `2`，需根据集群节点数量与服务器性能进行调整

### nfs

NFS 配置

#### server

NFS 服务器地址

#### path

NFS 服务器共享路径，需确保该路径可读写

#### storage

创建的 NFS PVC 存储空间大小，单位 `Gi`、`Mi`，默认 `10Gi`，需根据实际情况进行调整
