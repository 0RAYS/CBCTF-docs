---
sidebar_position: 2
---

# 快速上手

CBCTF 通过 Kubernetes 的方式进行部署，推荐使用 [k3s](https://k3s.io/) 作为集群环境。相关的部署细节请参考 `部署`。

## 配置

1. 下载 `release二进制文件` 或 克隆仓库并编译

   ```bash
   git clone https://github.com/0RAYS/CBCTF.git
   cd CBCTF
   go build
   ```

2. 初次运行将自动生成 `config.yaml` 配置文件。请根据实际情况修改。具体字段解释见 `settings`。

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
         image: snowdreamtech/frpc:latest
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
   
## 运行

1. 确保 `config.yaml` 配置正确后，运行 `CBCTF` 对集群进行初始化配置（注：并非安装集群，而是对程序所需要的资源进行初始化。安装集群请参考 `部署`）。

   ```bash
   ./CBCTF k8s init
   ```

   此时，将会依据配置项创建对应的 `namespace` `pv` `pvc` `network-attachment-define` 等资源，资源名称以 `namespace` 作为前缀。

2. 根据上述命令最终的输出结果，手动挂载 `NFS` 共享目录至配置项中的 `path` 位置。

   ```bash
   sudo mount -t nfs <NFS_SERVER_IP>:/mnt/data ./data
   ```

3. 启动服务

   ```bash
   ./CBCTF
   ```

4. 配置 `systemd` 进行后台运行（可选）

   ```ini
   [Unit]
   Description=CBCTF Service
   After=network.target

   [Service]
   Type=simple
   User=user
   WorkingDirectory=/path/to/CBCTF
   ExecStart=/path/to/CBCTF/CBCTF
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=multi-user.target
   ```

    启动服务
    
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start cbctf
    sudo systemctl enable cbctf
    ```
   
5. 配置开机自动挂载 `NFS` 共享目录（可选）

   编辑 `/etc/fstab` 文件，添加如下内容：
   ```ini
   <NFS_SERVER_IP>:/mnt/data /path/to/CBCTF/data nfs defaults 0 0
   ```
   
## 初始化管理员

当数据库中不存在管理员用户时，运行服务将会自动进行初始化管理员用户，相关账号密码将会打印在日志中，请及时进行修改，管理员登录地址为 `https://host/platform/#/admin/login`。

   ```log
   Init Admin: Admin{ name: admin, password: [uuid], email: admin@0rays.club}
   ```
