# Kubernetes 学习提纲

## 说明

各章独立成篇，建议顺着目录往下看。正文只写概念、命令和资源清单，不放截图，也不附带外部领取入口。

## 介绍

这条主线从“集群里有哪些角色”讲到日常运维：先把单控制面用两种装法跑起来，再写清单、管 Pod 和控制器，然后补权限、存储、监控，以及多控制面。图形界面和一键脚本放在后面，用来对照手工安装时见过的对象。

## 目录

- [Kubernetes 简介](./1_Kubernetes简介/README.md)
- [搭建前的主机准备](./2_搭建K8S集群前置知识/README.md)
- [用 kubeadm 安装](./3_使用kubeadm方式搭建K8S集群/README.md)
- [用二进制包安装](./4_使用二进制方式搭建K8S集群/README.md)
- [两种装法对照](./5_Kubeadm和二进制方式对比/README.md)
- [kubectl 日常命令](./6_Kubernetes集群管理工具kubectl/README.md)
- [资源清单怎么写](./7_Kubernetes集群YAML文件详解/README.md)
- [最小调度单元 Pod](./8_Kubernetes核心技术Pod/README.md)
- [工作负载 Controller](./9_Kubernetes核心技术Controller/README.md)
- [服务发现 Service](./10_Kubernetes核心技术Service/README.md)
- [控制器分类型](./11_Kubernetes控制器Controller详解/README.md)
- [配置与密钥](./12_Kubernetes配置管理/README.md)
- [访问控制与安全](./13_Kubernetes集群安全机制/README.md)
- [七层入口 Ingress](./14_Kubernetes核心技术Ingress/README.md)
- [包管理 Helm](./15_Kubernetes核心技术Helm/README.md)
- [卷与持久化](./16_Kubernetes持久化存储/README.md)
- [指标与监控面板](./17_Kubernetes集群资源监控/README.md)
- [多控制面高可用](./18_Kubernetes搭建高可用集群/README.md)
- [把业务镜像交到集群](./19_Kubernetes容器交付介绍/README.md)
- [脚本方式安装](./30_使用kubeadm-ha脚本一键安装K8S/README.md)
- [KubeSphere 控制台](./31_Kubernetes可视化界面kubesphere/README.md)
- [指定默认存储类](./32_Kubernetes配置默认存储类/README.md)

## 扩展

下面几篇不改变对象模型，只是换一种安装或观察方式：

- [一次性脚本安装](./30_使用kubeadm-ha脚本一键安装K8S/README.md)
- [KubeSphere 图形控制台](./31_Kubernetes可视化界面kubesphere/README.md)
- [默认 StorageClass](./32_Kubernetes配置默认存储类/README.md)
- [用 Rancher 接管现有集群](./50_使用Rancher搭建Kubernetes集群/README.md)
