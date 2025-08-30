---
sidebar_position: 1
---

# 集群部署

CBCTF 支持在 Kubernetes 集群中部署，推荐使用 K3S 进行部署，更方便便捷。本文档不作为 Kubernetes 细致的使用说明，仅涉及 CBCTF 平台部署内容，具体 Kubernetes 使用请参考[官方文档](https://kubernetes.io/zh-cn/docs/home/)

## 前提条件

准备多台 Linux 服务器，使用 Ubuntu 22.04 或更高版本，分为一台 Master 节点与多台 Worker 节点

**以 ESXI 虚拟机为例，`vSwitch` 需要开启以下内容；因不支持自定义 macvlan，云服务集群无法使用 [VPC 网络](/docs/features/vpc#vpc网络环境) 部署靶机**

![vswitch.png](img/vswitch.png)

### 调整网卡配置

须使所有节点的主网卡（访问外部网络的网卡）名称相同，例如均为 `eth0`，否则可能出现部分节点负载的靶机无法访问外部网络

自行上网查找资料修改

### 安装 NFS 客户端

安装 nfs 相关软件包，以支持 Kubernetes NFS 挂载

```bash
sudo apt update
sudo apt install -y nfs-common
```

## 安装 [K3S](https://docs.k3s.io/zh/quick-start)

1. Master 节点安装 K3S，需取消默认安装的 flannel 网络插件

   ```bash
   curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh - --flannel-backend=none --disable-network-policy 
   ```

2. Worker 节点安装 K3S，需取消默认安装的 flannel 网络插件，并指定连接至 Master。`mynodetoken` 值位于 Master 节点 `/var/lib/rancher/k3s/server/node-token`

   ```bash
   curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh - --flannel-backend=none --disable-network-policy 
   ```

3. 集群凭证位于 `/etc/rancher/k3s/k3s.yaml`，推荐使用[Lens](https://k8slens.dev/)对集群进行管理

## 安装 [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni/)

### 推荐

推荐使用 `Thin Plugin` 无需额外配置

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
```

### 不推荐

如需使用 `Thick Pulgin`，需注意一下问题

- [OOMKilled](https://github.com/k8snetworkplumbingwg/multus-cni/issues/1346)
- [Text file busy](https://github.com/k8snetworkplumbingwg/multus-cni/issues/1221)

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
```

解决方案参考

- [Issue #1346](https://github.com/k8snetworkplumbingwg/multus-cni/issues/1346#issuecomment-2644110944)
- [PR #1213](https://github.com/k8snetworkplumbingwg/multus-cni/pull/1213)

## 安装 [Kube-OVN](https://kubeovn.github.io/docs/stable/)

于 Master 节点执行，如需做额外配置，请自行参考[配置参数](https://kubeovn.github.io/docs/stable/start/one-step-install/#_4)修改

```bash
wget https://raw.githubusercontent.com/kubeovn/kube-ovn/refs/tags/v1.14.5/dist/images/install.sh
bash install.sh
```

## 配置 StorageClass

仅 K3S 需要配置，K8S 可跳过此步骤

因需使用 NFS 作为 Pod 间文件同步，需要使用 WriteMany 类型的存储卷，K3S 默认的 StorageClass 不支持该类型存储卷，取消其作为默认 StorageClass，由 `CBCTF` 主程序初始化时创建 PV PVC 替代

修改 `local-path` StorageClass，取消其作为默认 StorageClass

```yaml
storageclass.kubernetes.io/is-default-class: 'false'
```
