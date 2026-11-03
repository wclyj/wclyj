# 搭建集群前要先定的事

## 拓扑怎么选

### 单个控制面

一台控制面带着若干工作节点。实验够用：证书和 etcd 只有一份。控制面宕机后，已运行的 Pod 还在，但无法再调度或更新对象。

### 多个控制面

控制面做成奇数台，常见是 3 台，前面加负载均衡。工作节点访问均衡地址，而不是某一台的 IP。任意一台控制面维护时，API 仍然可用。

## 机器规格

实验可以收紧：控制面 2 核 4G、磁盘 40G；工作节点 4 核 8G、磁盘 80G。

生产按副本和镜像体积留余量。起步可按控制面 8 核 16G、磁盘 200G，工作节点 16 核 64G、磁盘 500G，监控和 etcd 磁盘另算。etcd 对延迟敏感，不要和控制面日志挤在同一块慢速盘上。

## 两条落地路线

### kubeadm

官方引导工具。`kubeadm init` 拉起第一台控制面，`kubeadm join` 加入其余节点。证书和静态 Pod 清单由工具生成。

文档：<https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>

### 二进制自行拼装

从发行包取出 apiserver、controller-manager、scheduler、kubelet、kube-proxy，再单独准备 etcd，用 systemd 逐个托管。步骤长，但参数和证书路径都在自己手里。

kubeadm 把这些步骤折进两条命令，排障时要在 kubelet、静态 Pod 和 kubeadm 日志之间来回看。想先跑起来用 kubeadm；想摸清协作过程用二进制。

## 动手前的检查单

- 生产选用仍在维护的系统。教程常见的 CentOS 7 已停更，实验可沿用，正式环境更宜 Rocky Linux 或 Ubuntu LTS。
- 每台至少 2 个 CPU、2G 内存，控制面磁盘不少于 30G。内存再低时 kubelet 容易被挤掉，节点反复 NotReady。
- 节点内网互通，主机名可解析，时间同步。
- 能访问镜像仓库，或事先把控制面镜像导入内网。
- 关闭 swap。kubelet 默认拒绝在开启 swap 的机器上启动。

工作节点加入前要先装好 CNI，否则新节点会停在 NotReady。
