# 用 kubeadm 搭建集群

kubeadm 是官方提供的安装辅助程序，负责证书、静态 Pod 和控制面组件的摆放。它把“装一套能用的集群”收成两步：

```bash
kubeadm init
kubeadm join <控制面地址:端口>
```

第一条只在第一台控制面执行。工作节点执行第二条，地址和令牌以 init 的实际输出为准。

## 步骤

1. 准备三台 CentOS 7 虚拟机
2. 三台做同样的主机初始化
3. 安装容器运行时、kubelet、kubeadm、kubectl
4. 在控制面执行 `kubeadm init`
5. 两个工作节点执行 `kubeadm join`
6. 安装网络插件，节点才会从 NotReady 变成 Ready
7. 跑一个 nginx 工作负载，并用节点端口从集群外访问

## 安装条件

- 64 位 CentOS 7
- 每台至少 2 核 CPU、2 GB 内存、30 GB 磁盘；控制面核数少于 2 时，预检会直接失败
- 能拉取镜像；不能上网时要事先 `docker load` 或 `ctr import`
- 关闭 swap

## 机器规划

| 角色 | 主机名 | IP |
| --- | --- | --- |
| 控制面 | ctrl-1 | 10.20.8.11 |
| 工作节点 | worker-a | 10.20.8.21 |
| 工作节点 | worker-b | 10.20.8.22 |

下面的初始化三台都做。主机名按行替换。

```bash
systemctl disable --now firewalld
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
hostnamectl set-hostname ctrl-1
cat >> /etc/hosts << 'EOF'
10.20.8.11 ctrl-1
10.20.8.21 worker-a
10.20.8.22 worker-b
EOF
cat > /etc/sysctl.d/k8s.conf << 'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
modprobe br_netfilter
sysctl --system
yum install -y chrony
systemctl enable --now chronyd
```

`worker-a` 和 `worker-b` 只改 `hostnamectl` 那一行。hosts 三台保持一致。

## 安装运行时和 kubeadm

示例仍用 Docker 作为运行时，便于对照 `docker ps`。生产环境更常见的是 containerd，初始化参数要改成对应的 CRI 套接字。

### 安装 Docker

写入仓库文件后安装。镜像加速地址换成你自己的，不要照抄别人的专属域名。

```bash
cat > /etc/yum.repos.d/docker-ce.repo << 'EOF'
[docker-ce]
name=Docker CE
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
EOF
yum install -y docker-ce
systemctl enable --now docker
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://mirror.example.net"]
}
EOF
systemctl restart docker
docker info | grep -i cgroup
```

cgroup 驱动应显示 systemd，和 kubelet 默认一致，减少后面预检告警。

### Kubernetes 软件源

```bash
cat > /etc/yum.repos.d/kubernetes.repo << 'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF
```

### 安装指定版本

仓库里的最新版经常超前于你准备好的镜像。这里固定 1.23.6。

```bash
yum install -y kubelet-1.23.6 kubeadm-1.23.6 kubectl-1.23.6
systemctl enable kubelet
```

此时 kubelet 会不断重启，因为还没有 `/var/lib/kubelet/config.yaml`。等 init 或 join 完成后再看它的状态。

## 初始化控制面

只在 `10.20.8.11` 执行。`--image-repository` 用来避开默认的 `registry.k8s.io` 拉取超时。Pod 网段必须和后面的网络插件一致，这里用 `10.80.0.0/16`。

```bash
kubeadm init \
  --apiserver-advertise-address=10.20.8.11 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.23.6 \
  --service-cidr=10.96.0.0/16 \
  --pod-network-cidr=10.80.0.0/16
```

命令会先拉 pause、etcd、API、调度器、控制器管理器和 CoreDNS 的镜像，耗时主要在这里。另开终端执行 `docker images`，能看到 `registry.aliyuncs.com/google_containers` 前缀的镜像逐个落地。预检通过后才会写证书和静态 Pod 清单。成功时末尾打印两段 `kubeadm join`：带 `--control-plane` 的留给以后加控制面，普通那段才是工作节点用的。令牌和 ca 哈希每人不同，抄自己的输出，不要借用文档里的示例值。

接着只在控制面配置 kubectl：

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

`ctrl-1` 会显示 NotReady。这是预期现象：网络插件还没装，节点之间的 Pod 网段不通，就绪条件不会满足。此时 `kubectl get pods -n kube-system` 里，etcd 和 API 应为 Running，CoreDNS 可能还在 ContainerCreating，等网络插件就绪后再变成 Running。

## 加入工作节点

在 `worker-a` 和 `worker-b` 上粘贴 init 输出的 join。下面只是格式示例，令牌不能复用：

```bash
kubeadm join 10.20.8.11:6443 --token a7k2mq.9pl4c0sampletoken \
  --discovery-token-ca-cert-hash sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

默认令牌大约 24 小时失效。过期后在控制面重新生成：

```bash
kubeadm token create --print-join-command
```

两台都加入后，回到 `ctrl-1`：

```bash
kubectl get nodes
```

三行都应在，状态仍可能是 NotReady。

## 安装网络插件

没有 CNI 时，跨节点通信和节点就绪都不会完成。下面使用 Flannel 0.22.3，并让它的 Pod 网段与 init 时的 `--pod-network-cidr` 相同。若仓库访问慢，可把清单下载到控制面再改，不必每次都直接 `kubectl apply` 远程地址。

```bash
curl -fsSL -o kube-flannel.yml \
  https://raw.githubusercontent.com/flannel-io/flannel/v0.22.3/Documentation/kube-flannel.yml
```

打开文件，确认 `net-conf.json` 里的 Network 是 `10.80.0.0/16`。若仍是 `10.244.0.0/16`，改成与 init 一致后再应用。镜像若拉不动，可在三台先导入同版本的 flannel 镜像，并把清单里的镜像名改成你导入的那个。

```bash
kubectl apply -f kube-flannel.yml
kubectl -n kube-flannel get pods
kubectl get nodes
```

flannel Pod 为 Running 之后，三个节点变为 Ready。某个节点长期 NotReady 时，先在控制面把它移出，再在该节点重置并重新加入：

```bash
kubectl drain worker-a --ignore-daemonsets --delete-emptydir-data
kubectl delete node worker-a
```

在 `worker-a` 上：

```bash
kubeadm reset -f
```

然后用新的 join 命令加回。只在一台机器上导入镜像、其它机器没有时，常见表现就是这台节点的 flannel 起不来。

## 验证

```bash
kubectl create deployment web --image=nginx:1.25.4
kubectl get pods -o wide
kubectl expose deployment web --port=80 --type=NodePort
kubectl get svc web
```

Pod 为 Running 后，Service 会分配一个 30000 以上的节点端口，例如 31280。在浏览器访问 `http://10.20.8.21:31280/`。能打开欢迎页，说明调度、网络和 NodePort 都通了。这是单控制面：`ctrl-1` 关机后，已有 Pod 可能还在，但新建对象和故障转移会停住。多控制面的接法见高可用那一章。欢迎页只证明链路通，业务镜像、配额和权限仍按后面的章节单独配置。

## 排错

### 预检报 CPU 不足

```text
[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
```

虚拟机只分了 1 核。关机后把控制面调到 2 核再 init，不要用忽略预检的方式带过去。

### 预检报 swap 仍开启

```text
[ERROR Swap]: running with swap on is not supported
```

执行 `swapoff -a`，并确认 `/etc/fstab` 里的 swap 行已经注释。只做临时关闭时，重启后又会失败。

### join 时连不上 kubelet 健康检查

```text
dial tcp [::1]:10248: connect: connection refused
```

kubelet 没在听 10248。先看 `systemctl status kubelet` 和 `journalctl -u kubelet`。若是 cgroup 驱动和 Docker 不一致，回到前面的 `daemon.json`，把驱动改成 systemd，重启 Docker 和 kubelet。实验环境确认要放过 swap 检查时，再在 kubelet 的额外参数里加 `--fail-swap-on=false`，然后 `systemctl daemon-reload`。控制面若已经 init 失败，执行 `kubeadm reset -f`，删掉 `$HOME/.kube` 后重新 init。

### kubectl 报证书签发者不被信任

```text
x509: certificate signed by unknown authority
```

多半是 `kubeadm reset` 之后仍在使用旧的 `$HOME/.kube/config`。删掉该目录，再从新的 `/etc/kubernetes/admin.conf` 复制一份。不要把两代 admin.conf 拼在同一个文件里。

### yum 一直等锁

```text
Another app is currently holding the yum lock
```

有另一次 yum 或系统更新还在跑。看 `/var/run/yum.pid` 对应的进程，等它结束再装。不要直接删锁文件，除非确认那个进程已经不存在。

### join 报 ip_forward 不是 1

```text
[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: contents are not set to 1
```

内核没有打开转发，跨节点的 Pod 流量过不去。写入并立即生效：

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

同时保留前面 `k8s.conf` 里的 `net.ipv4.ip_forward = 1`，否则重启后预检会再失败。改完重新执行 join。
