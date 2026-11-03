# 用二进制包搭建集群

## 说明

这一章只覆盖动手前的主机条件和证书工具。kube-apiserver、控制器管理器、调度器、kubelet 的 systemd 单元和参数文件还没展开，不能按本章独立装完一整套集群。需要先跑通时，用 kubeadm 那一章。

## 准备工作

二进制安装没有向导，机器要先满足：

- 系统为 CentOS 7 的 64 位版本
- 每台至少 2 核 CPU、2 GB 内存、30 GB 磁盘；控制面建议再留余量给 etcd
- 节点之间网络互通，主机名能互相解析
- 能访问外网以便下载发布包；隔离环境则提前把二进制和镜像拷进去
- 关闭 swap，kubelet 默认拒绝在开启 swap 的机器上启动

## 步骤

整条链路按这个顺序，后面几步在本章没有展开：

1. 准备虚拟机并装好系统
2. 统一做主机初始化
3. 为 etcd 和 API 进程签发证书
4. 先把 etcd 组成集群
5. 在控制面放 kube-apiserver、kube-controller-manager、kube-scheduler
6. 在工作节点放 kubelet、kube-proxy 和容器运行时
7. 最后安装网络插件，节点才会变成 Ready

## 准备虚拟机

示例只用两台，etcd 若要容忍一台故障，正式环境应扩成 3 或 5 台，不能把数据只放在控制面本机的单节点上。

| 主机名 | IP |
| --- | --- |
| bin-ctrl | 10.30.6.11 |
| bin-worker | 10.30.6.21 |

## 操作系统初始化

两台都执行。主机名按角色替换。

```bash
systemctl disable --now firewalld
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
hostnamectl set-hostname bin-ctrl
cat >> /etc/hosts << 'EOF'
10.30.6.11 bin-ctrl
10.30.6.21 bin-worker
EOF
cat > /etc/sysctl.d/k8s.conf << 'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system
yum install -y chrony
systemctl enable --now chronyd
```

`bin-worker` 上把 `hostnamectl` 的参数换成自己的名字。hosts 建议两台写成一样，避免一边能解析、一边不能。

## 部署 etcd

etcd 保存集群对象。单副本在这台机器宕机时，整个控制面失去读写能力。3 台可容忍 1 台故障，5 台可容忍 2 台。先准备证书，再分发二进制和数据目录。

### 为什么要自签

浏览器里的 HTTPS 证书用来证明“这个网站就是它声称的那一个”。集群内部也一样：API 进程、etcd 和 kubelet 互相调用时都要校验对方。没有被同一套 CA 签过的证书，调用会在 TLS 握手阶段失败。

二进制安装不会替你生成这些文件，需要自己当 CA。可以把它想成公司自己印的门卡：员工卡能进机房，访客卡只能进前台。集群 CA 签给控制面组件的证书权限大，签给某个用户的客户端证书则只代表身份，能做什么仍由 RBAC 决定。

外部用户、节点和管理员可以共用同一个集群 CA，也可以拆成独立 CA。拆开后，泄漏一张节点证书不会顺便伪造管理员。

### 准备 cfssl

cfssl 用 JSON 描述证书请求，比直接记一长串 openssl 参数更容易复现。在 `bin-ctrl` 上下载 R1.2 的三个可执行文件并放到 PATH：

```bash
curl -L -o /tmp/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -L -o /tmp/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
curl -L -o /tmp/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x /tmp/cfssl /tmp/cfssljson /tmp/cfssl-certinfo
mv /tmp/cfssl /usr/local/bin/cfssl
mv /tmp/cfssljson /usr/local/bin/cfssljson
mv /tmp/cfssl-certinfo /usr/local/bin/cfssl-certinfo
cfssl version
```

能打出版本号之后，再写 CA 的 JSON 并生成 etcd 与 API 使用的证书。那部分和各进程的单元文件一起，留到把本章补全时再写。
