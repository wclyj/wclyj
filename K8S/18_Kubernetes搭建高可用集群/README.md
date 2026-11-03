# 搭建高可用集群

## 前言

只有一台控制面时，这台机器停机，工作节点上已经跑着的容器未必立刻消失，但新建 Pod、扩容、证书轮换和控制器纠偏都会停住。kubectl 也连不上唯一的 API 地址。管理面本身成了单点。

## 要补上什么

至少两台控制面同时提供 API。工作节点和运维入口前面再放一层负载均衡，做两件事：把请求分到存活的 API 进程，并在某一台控制面失效时把它摘掉。

对外只公布一个虚拟地址。这个地址不固定在某块网卡上，由保活协议在控制面之间漂移。客户端、kubeadm 的 `controlPlaneEndpoint` 和 join 命令都写这个地址，不写某一台机器的物理 IP。

## 组件怎么分工

- keepalived：在两台控制面之间选主，把虚拟地址绑到主节点的网卡，并检查本机负载均衡进程是否还在
- HAProxy：接受虚拟地址上的流量，再转发到两台机器真实的 6443
- 每台控制面仍各自运行 kube-apiserver、kube-controller-manager、kube-scheduler 和一份本地 etcd

控制器管理器和调度器内部会做选主，多副本同时存在是预期状态，不是装重了。etcd 在这个实验里用 kubeadm 的本地栈，每台控制面一份成员。

## 节点分工

两台控制面加一台工作节点。虚拟地址 `10.20.8.20` 不配置在任何一台的静态地址里。

| 主机名 | 地址 | 要装的内容 |
| --- | --- | --- |
| ctrl-a | 10.20.8.21 | keepalived、HAProxy、控制面 |
| ctrl-b | 10.20.8.22 | 同上，keepalived 为备 |
| worker-c | 10.20.8.31 | 运行时和 kubelet，最后 join |

虚拟地址的主机名记为 `k8s-api`，解析到 `10.20.8.20`。

## 初始化三台机器

```bash
systemctl disable --now firewalld
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
hostnamectl set-hostname ctrl-a
cat >> /etc/hosts << 'EOF'
10.20.8.20 k8s-api
10.20.8.21 ctrl-a
10.20.8.22 ctrl-b
10.20.8.31 worker-c
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

`ctrl-b`、`worker-c` 把主机名换成自己的。hosts 三份内容相同。

## 部署 keepalived

在 `ctrl-a` 和 `ctrl-b` 安装。工作节点不参与虚拟地址选举。

### 安装

```bash
yum install -y keepalived
```

### ctrl-a 作为优先主节点

网卡名用 `ip -br link` 核对，下面按 `ens192` 书写。`virtual_router_id` 两台必须相同，且不要和局域网里已有的 VRRP 组冲突。优先级主高备低。

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id ctrl-a
}
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -30
    fall 3
    rise 2
}
vrrp_instance API_VIP {
    state MASTER
    interface ens192
    virtual_router_id 77
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass lab-vip-77
    }
    virtual_ipaddress {
        10.20.8.20/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### ctrl-b 作为备节点

除了 `state`、`priority` 和 `router_id`，其余与主节点一致，尤其是路由组和口令。

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id ctrl-b
}
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -30
    fall 3
    rise 2
}
vrrp_instance API_VIP {
    state BACKUP
    interface ens192
    virtual_router_id 77
    priority 120
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass lab-vip-77
    }
    virtual_ipaddress {
        10.20.8.20/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### 启动后确认虚拟地址

两台都执行：

```bash
systemctl enable --now keepalived
systemctl status keepalived --no-pager
ip -4 addr show ens192
```

正常时只有 `ctrl-a` 的网卡上出现 `10.20.8.20`。把 `ctrl-a` 的 keepalived 停掉，地址应在数秒内出现在 `ctrl-b`。确认后再把 `ctrl-a` 拉起来。若两边同时带着这个地址，先核对 `virtual_router_id` 是否写成了两个值。

## 部署 HAProxy

负载均衡跑在两台控制面上，把到达虚拟地址的 API 流量转到本机或对端的 6443。不要把它理解成给业务 NodePort 做转发，那是后面 Ingress 或另一层均衡器的事。

### 安装

```bash
yum install -y haproxy
systemctl enable --now haproxy
```

默认配置先别急着看端口。写完下面的文件并重载后，才应该听到 18443。

### 两台写成同一份

前端绑定 18443，后端是两台控制面的真实 API 端口。统计页放在 19090，口令只用于实验。

```bash
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log 127.0.0.1 local2
    maxconn 5000
    user haproxy
    group haproxy
    daemon

defaults
    mode tcp
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend api_front
    bind *:18443
    default_backend api_back

backend api_back
    balance roundrobin
    option tcp-check
    server ctrl-a 10.20.8.21:6443 check
    server ctrl-b 10.20.8.22:6443 check

listen stats
    bind *:19090
    mode http
    stats enable
    stats uri /stats
    stats auth ops:lab-stats-19090
EOF
systemctl restart haproxy
ss -lnt | grep -E '18443|19090'
```

18443 还没有 kube-apiserver 时，健康检查会先标红，这是正常的。init 完成后再看统计页 `http://10.20.8.21:19090/stats`。

## 安装运行时和 Kubernetes 组件

三台都装。运行时仍用 Docker，并固定 cgroup 驱动。

```bash
yum install -y docker-ce
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://mirror.example.net"]
}
EOF
systemctl enable --now docker
cat > /etc/yum.repos.d/kubernetes.repo << 'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF
yum install -y kubelet-1.22.17 kubeadm-1.22.17 kubectl-1.22.17
systemctl enable kubelet
```

Docker 的仓库文件若还没配，沿用单控制面那一章的 docker-ce 源即可，版本不要和 kubelet 各写各的。

## 初始化第一台控制面

只在当前持有虚拟地址的 `ctrl-a` 上做。`controlPlaneEndpoint` 写虚拟主机名和 HAProxy 端口，不要写 `10.20.8.21:6443`。

```bash
mkdir -p /root/ha
cat > /root/ha/kubeadm-ha.yaml << 'EOF'
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.22.17
controlPlaneEndpoint: "k8s-api:18443"
imageRepository: registry.aliyuncs.com/google_containers
networking:
  podSubnet: 10.81.0.0/16
  serviceSubnet: 10.97.0.0/16
  dnsDomain: cluster.local
apiServer:
  certSANs:
    - k8s-api
    - ctrl-a
    - ctrl-b
    - 10.20.8.20
    - 10.20.8.21
    - 10.20.8.22
    - 127.0.0.1
etcd:
  local:
    dataDir: /var/lib/etcd
EOF
kubeadm init --config /root/ha/kubeadm-ha.yaml --upload-certs
```

`--upload-certs` 会给出一份短期证书密钥。第二台控制面加入时要带上它；过期就在第一台执行 `kubeadm init phase upload-certs --upload-certs` 重新生成。

配置 kubectl：

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
kubectl get pods -n kube-system
```

把终端里带 `--control-plane` 的 join 整段保存下来。没有这个标记的那一段才是给工作节点用的。

## 安装网络插件

在 `ctrl-a` 下载与 Pod 网段匹配的 Flannel 清单。把 Network 改成 `10.81.0.0/16` 再应用。

```bash
curl -fsSL -o /root/ha/kube-flannel.yml \
  https://raw.githubusercontent.com/flannel-io/flannel/v0.22.3/Documentation/kube-flannel.yml
kubectl apply -f /root/ha/kube-flannel.yml
kubectl -n kube-flannel get pods
```

## 第二台控制面加入

先把集群 CA 和前端代理 CA 拷到 `ctrl-b`。etcd 的 CA 也要在，否则第二份本地 etcd 成员签不出证书。业务密钥和各节点自己的 apiserver 证书不要从第一台整目录覆盖过去。

```bash
ssh root@10.20.8.22 'mkdir -p /etc/kubernetes/pki/etcd'
scp /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/ca.key \
    /etc/kubernetes/pki/sa.key /etc/kubernetes/pki/sa.pub \
    /etc/kubernetes/pki/front-proxy-ca.crt /etc/kubernetes/pki/front-proxy-ca.key \
    root@10.20.8.22:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/etcd/ca.key \
    root@10.20.8.22:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf root@10.20.8.22:/etc/kubernetes/admin.conf
```

在 `ctrl-b` 执行 init 输出里带 `--control-plane` 和 `--certificate-key` 的命令。格式类似：

```bash
kubeadm join k8s-api:18443 --token sample.tokenvalue \
  --discovery-token-ca-cert-hash sha256:abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789 \
  --control-plane \
  --certificate-key 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

回到任一控制面：

```bash
kubectl get nodes
kubectl get pods -A
```

应看到 `ctrl-a` 和 `ctrl-b` 都是 control-plane。`kube-system` 里两套 API 和 etcd Pod 都为 Running 后，再加工作节点。

## 加入工作节点

在 `worker-c` 执行不带 `--control-plane` 的 join，地址仍然是 `k8s-api:18443`。

```bash
kubeadm join k8s-api:18443 --token sample.tokenvalue \
  --discovery-token-ca-cert-hash sha256:abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789
```

新节点加入后确认 Flannel 在该节点也有 Pod：

```bash
kubectl get nodes
kubectl -n kube-flannel get pods -o wide
```

若 `worker-c` 上的 Flannel 缺失，再对清单执行一次 `kubectl apply`，DaemonSet 会补齐。

## 验证

```bash
kubectl create deployment web --image=nginx:1.25.4
kubectl expose deployment web --port=80 --type=NodePort
kubectl get pod,svc
```

用 `worker-c` 的地址加节点端口访问欢迎页。然后停掉 `ctrl-a` 上的 `keepalived`，确认虚拟地址漂到 `ctrl-b` 之后，`kubectl --server=https://k8s-api:18443 get nodes` 仍然返回。把 `ctrl-a` 恢复后，两台 API 应重新都出现在 HAProxy 的统计页里。这只能说明入口没有绑死在一台机器上；etcd 只有两成员时容忍不了一台以上的同时故障，正式环境应把成员数做成奇数。
