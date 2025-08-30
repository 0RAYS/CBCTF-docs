---
sidebar_position: 3
---

# 网络环境

针对容器题目，支持两种不同的网络环境，不同环境中容器之间通信方式不同

## 默认网络环境

以下述 `docker-compose.yaml` 为例，对外暴露的端口为 `web` 容器的 `5000` 端口，`web` 容器中包含两个 flag，`db` 容器为 MySQL 数据库

```yaml
services:
 web:
  image: docker.0rays.club/cbctf/test-web:20250717
  ports:
    - "5000:5000"
  environment:
    - FLAG_0=uuid{}
    - FLAG_1=dynamic{this_is_a_dynamic_flag}

 db:
   image: docker.0rays.club/library/mysql:8.0
   environment:
     MYSQL_ROOT_PASSWORD: example
     MYSQL_DATABASE: testdb
```

在常规 docker 环境中，将会启动两个容器，分别为 `web` 与 `db`，两个容器将会被加入到同一个默认网络中，容器之间可以通过容器名称进行通信，例如 `web` 容器可以通过 `db:3306` 访问 `db` 容器。

**但是在 Kubernetes 环境中，两个容器将会被加入到同一个 Pod 中。此时两个容器公用相同的网络命名空间，`web` 容器将使用 `localhost:3306` 访问 `db` 容器。**

**出题人需要注意在编写代码时，因使用本地地址访问其他容器的服务。**

## VPC网络环境

Virtual Private Cloud，允许出题人构建一个独立的虚拟网络环境，自定义网络结构，子网划分

以下述 `docker-compose.yaml` 为例，对外暴露的端口为 `web1` 容器的 `5000` 端口，`web1`、`web2`、`web3` 三个容器分别包含一个 flag，三个容器分别加入到两个不同的子网中

```yaml
services:
  web1:
    ports:
      - "5000:5000"
    environment:
      FLAG_1: uuid{}
    image: docker.0rays.club/cbctf/test-web:20250717
    networks:
      network1:
        ipv4_address: 192.168.0.2
      network2:
        ipv4_address: 192.168.1.2
  web2:
    environment:
      FLAG_2: uuid{}
    image: docker.0rays.club/cbctf/test-web:20250717
    networks:
      network2:
        ipv4_address: 192.168.1.3
      network3:
        ipv4_address: 192.168.2.3
  web3:
    environment:
      FLAG_3: uuid{}
    image: docker.0rays.club/cbctf/test-web:20250717
    networks:
      network3:
        ipv4_address: 192.168.2.4
networks:
  network1:
    ipam:
      config:
        - subnet: 192.168.0.0/24
          gateway: 192.168.0.1
    external: true
  network2:
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1
  network3:
    ipam:
      config:
        - subnet: 192.168.2.0/24
          gateway: 192.168.2.1
```

此时，`web1` 通过 `192.168.1.3` 访问 `web2`，`web2` 通过 `192.168.2.4` 访问 `web3`，`web1` 无法直接访问 `web3`。

同时因 `network1` exiternal 设置为 `true`，表示该网络允许访问外网，其余两个网络无法访问外部网络，仅允许内网通信
