# Kubernetes 入门：从单机容器到集群

容器解决了“这一份环境能在别的机器上跑起来”。机器一多，就要回答另一组问题：副本挂了谁来拉起，流量打到哪一个地址，滚动替换时怎样不停完，配置和口令放在哪里。Kubernetes 就是这层编排。

## 它负责哪些事

- 发现与分流。可以为 Pod 分配稳定的虚拟地址，也可以给一组 Pod 一个 DNS 名字。流量变大时，由 Service 把连接分到现存副本，而不是让调用方自己挑 IP。
- 存储编排。Pod 声明需要一块盘，集群按 StorageClass 去挂本地盘、网络盘或云盘，调度器知道这块盘在哪台节点上。
- 按期望状态发布。你写明要三个副本、用哪一个镜像标签。实际数量少了就补，滚动更新失败可以退回上一个修订。金丝雀也是先放一小部分新副本，再逐步放大。
- 按请求放置。每个容器可以声明需要多少 CPU 和内存。调度器据此避开已经挤满的节点，而不是只看节点上还有没有“空位”这种模糊说法。
- 自行恢复。容器退出会被按策略重启；节点失联后，控制器在宽限期过后把 Pod 安排到别处。未通过就绪探针的 Pod 不会被 Service 选中，因此还在启动的实例不会提前接流量。
- 配置与密钥。口令、令牌和普通配置存在独立对象里，挂进 Pod 时不必重新构建镜像，也不必写进仓库。

这些能力加在一起，就是一份可以伸缩、可以故障转移的运行框架。它不替代业务里的事务和补偿，只保证你声明的进程和网络形态被维持。

## 集群由什么组成

一套集群等于若干控制面节点加上若干工作节点，两边都可以是一台。实验常用一台控制面加两台工作节点。控制面决定“要发生什么”，工作节点负责“在本机把容器跑起来”。

### 控制面

控制面组件做全局决定：要不要补副本、新 Pod 落在哪台机器、节点失联后如何标记。为了省机器，安装工具通常把它们放在同一台，并且不在这台跑业务。要提高可用度，就在多台机器上各跑一份 apiserver，前面加负载均衡。做法见官方文档里 kubeadm 的高可用说明。

kube-apiserver 是唯一对外的入口。所有查询和变更都进这里，再由它写入 etcd。它可以水平扩展，多副本前面用负载均衡分摊请求。

etcd 保存集群对象。它是一致的键值存储，控制面的期望状态都在里面。需要单独做备份计划，只备份工作节点磁盘救不回集群对象。

kube-scheduler 盯着还没有 `nodeName` 的 Pod。它综合考虑请求量、污点与容忍、亲和性、数据所在位置，选出一台节点并写回绑定结果。调度器不负责拉镜像。

kube-controller-manager 把多种控制器编进同一个进程，逻辑上仍是多条循环：

- 节点控制器：节点停止心跳后改条件、发通知。
- Job 控制器：为一次性任务创建 Pod，完成后不再拉起。
- 端点控制器：把 Service 的选择器和当前 Pod 地址写成 Endpoints。
- 服务账号控制器：新命名空间创建时放上默认账号。

云控制器 cloud-controller-manager 只在对接公有云时需要。它含有厂商 API 的逻辑，例如向云平台确认节点是否已删除、配置路由、创建云负载均衡。笔记本或自建机房不部署它。和 kube-controller-manager 一样，它可以跑多个副本。其中节点控制器负责问云平台这台机器是否还在，路由控制器改底层路由，服务控制器维护云负载均衡的监听和后端。

### 工作节点

每台工作节点都跑 kubelet 和 kube-proxy，并安装容器运行时。

kubelet 接受本机应运行的 Pod 规格，调用 CRI 保证容器处于规格要求的状态。它不管理你用 `docker run` 私下拉起的容器。

kube-proxy 在节点上维护转发规则，让访问 Service 虚拟地址的流量到达 Pod。系统如果有包过滤能力，它就写规则；否则退化为自己转发。

一次创建的顺序可以串起来看：kubectl 把清单交给 apiserver；控制器补 Pod；调度器写上节点名；目标节点的 kubelet 看到绑定，调用运行时创建容器；kube-proxy 的规则使 Service 能把流量送进来。

## 用 kubeadm 把集群拉起来

先在每一台机器装好容器运行时，cgroup 驱动与 kubelet 一致。然后选一台做控制面，执行 `kubeadm init`。这一步会装上 apiserver、控制器管理器、调度器和 etcd。其余机器执行 `kubeadm join` 成为工作节点。kube-proxy 以 DaemonSet 的形式出现在每一台节点上，不是只存在于控制面。

### 安装前检查

- 使用 kubeadm 支持的 Linux。Debian 系和 Red Hat 系都有现成的软件源。
- 每台不少于 2 个 CPU、2G 内存。内存再低，系统进程和 kubelet 会互相抢，节点反复 NotReady。
- 节点之间内网互通。主机名、MAC、`product_uuid` 都不要重复，否则节点身份会冲突。
- 控制面与工作节点所需端口按官方清单放行，实验若暂时关了防火墙，收尾时要改回白名单。
- 关闭 swap，并且写进 fstab，防止重启后又打开。kubelet 默认在 swap 开启时拒绝启动。

三台示例机器：

| 角色 | 主机名 | 内网地址 |
| --- | --- | --- |
| 控制面 | cp-1 | 10.20.1.10 |
| 工作节点 | wk-1 | 10.20.1.21 |
| 工作节点 | wk-2 | 10.20.1.22 |

### 每一台都做的系统准备

```bash
hostnamectl set-hostname cp-1
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo swapoff -a
sudo sed -ri 's/.*swap.*/#&/' /etc/fstab

cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

工作节点把主机名换成 `wk-1`、`wk-2`。`br_netfilter` 让桥接流量也经过 iptables，否则跨节点的 Service 转发会丢包。

### 安装 kubelet、kubeadm、kubectl

软件源示例指向可访问的镜像站，并固定到 `1.28.8`。三台版本必须相同。

```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF

sudo yum install -y kubelet-1.28.8 kubeadm-1.28.8 kubectl-1.28.8
sudo systemctl enable --now kubelet
```

此时 kubelet 会周期性重启。它在等待 kubeadm 写出配置之前找不到静态 Pod 目录，这是预期现象，不是安装失败。

### 预先拉取控制面镜像

外网不稳定时，先在每一台把镜像拉到本地。准确列表以 `kubeadm config images list --kubernetes-version v1.28.8` 为准，下面只演示脚本形状，仓库前缀换成你已同步的内网仓库：

```bash
sudo tee ./pull-images.sh <<'EOF'
#!/bin/bash
set -euo pipefail
prefix=registry.lab.local/k8s
images=(
  kube-apiserver:v1.28.8
  kube-controller-manager:v1.28.8
  kube-scheduler:v1.28.8
  kube-proxy:v1.28.8
  pause:3.9
  etcd:3.5.9-0
  coredns:v1.10.1
)
for name in "${images[@]}"; do
  docker pull "${prefix}/${name}"
done
EOF
chmod +x ./pull-images.sh && ./pull-images.sh
```

若使用容器运行时是 containerd 而不是 Docker，把 `docker pull` 换成 `ctr` 或 `nerdctl`，并确认镜像进入了 kubelet 所使用的那个 namespace。

### 初始化控制面

所有节点把控制面地址记成一个名字，方便以后把单控制面换成负载均衡地址而不改工作节点的 join 参数：

```bash
echo '10.20.1.10 k8s-api.lab.local' >> /etc/hosts
```

只在 cp-1 上执行：

```bash
kubeadm init \
  --apiserver-advertise-address=10.20.1.10 \
  --control-plane-endpoint=k8s-api.lab.local \
  --image-repository=registry.lab.local/k8s \
  --kubernetes-version=v1.28.8 \
  --service-cidr=10.80.0.0/16 \
  --pod-network-cidr=10.90.0.0/16
```

四个网段不要重叠：节点内网是 `10.20.1.0/24`，Service 是 `10.80.0.0/16`，Pod 是 `10.90.0.0/16`。`--image-repository` 必须和前面拉取脚本的前缀一致，否则 init 仍会去公网拉取。

成功时输出会包含三部分：把管理员 kubeconfig 放到当前用户家目录、提醒安装网络插件、以及一条 `kubeadm join`。令牌和证书哈希每次不同，以终端当时打印的为准，不要从旧笔记里抄。

把 kubectl 指到新集群：

```bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
kubectl get nodes
```

cp-1 此时是 NotReady，因为还没有 CNI。也可以临时用 `export KUBECONFIG=/etc/kubernetes/admin.conf`，但那是 root 的文件，日常账号还是复制一份到家目录。

### 安装网络插件

以 Calico 为例，清单用与集群版本匹配的那一版，并确认里面的 Pod 网段等于 `10.90.0.0/16`。默认清单若仍是 `192.168.0.0/16`，要改成初始化时写的网段，否则跨节点通信用的地址和路由不一致。

```bash
curl -fsSL -o calico.yaml https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
kubectl apply -f calico.yaml
kubectl get pods -n kube-system -w
```

等到 `calico-node` 在每个节点上 Running，再看 `kubectl get nodes`，控制面应变为 Ready。

### 加入工作节点

在 wk-1 和 wk-2 上执行 init 输出的加入命令，形态如下（令牌请替换成你自己的）：

```bash
kubeadm join k8s-api.lab.local:6443 \
  --token <init-打印的令牌> \
  --discovery-token-ca-cert-hash sha256:<init-打印的哈希>
```

令牌过期后，在控制面重新生成一条，而不是把旧令牌的时间硬改长：

```bash
kubeadm token create --print-join-command
```

增加控制面节点时，使用 init 输出里带 `--control-plane` 的那条，并按提示把证书密钥拷过去。实验环境可以先不做高可用。

### 验收

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

三台都是 Ready，`kube-system` 里的 coredns、kube-proxy、calico 都 Running。再放一个不占用 80 端口的页面，确认调度和网络：

```bash
kubectl create deployment notice --image=httpd:2.4-alpine
kubectl expose deployment notice --port=8080 --target-port=80
kubectl rollout status deployment/notice
kubectl get pods -o wide
```

在集群内用一个临时 Pod 访问 Service 名字：

```bash
kubectl run curl-lab --image=curlimages/curl:8.5.0 -it --rm --restart=Never -- \
  curl -sI http://notice.default.svc.cluster.local:8080
```

能返回 HTTP 响应头，说明 CoreDNS 与 kube-proxy 都工作了。节点上若仍能使用 docker，`docker ps` 看到的是运行时视角的容器；集群视角始终以 `kubectl get pods` 为准，两者不是同一个名字空间。

## 打开 Dashboard

官方提供了一套只读查看加部分操作的网页，项目在 kubernetes/dashboard。版本要和集群匹配，1.28 可以先用 v2.7 这一线的兼容发布。不要把几年前的推荐清单原样套到新集群上，其中的 API 和命令行参数可能已经变了。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl get pods -n kubernetes-dashboard
```

上游清单会创建命名空间、ServiceAccount、Role、Deployment 和 ClusterIP Service。Pod 就绪后，把网页服务改成 NodePort，便于在实验网里用浏览器打开。生产更宜放在 Ingress 后面并启用登录，而不是把管理面端口暴露到公网。

```bash
kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard \
  -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8443,"nodePort":31443}]}}'
kubectl get svc -n kubernetes-dashboard
```

安全组放行 `31443`。浏览器访问 `https://10.20.1.21:31443`。证书是控制器自签的，浏览器会提示不受信任，实验环境可以临时继续。

默认账号没有集群管理员权限。单独做一份只用于实验的绑定，文件 `dash-admin.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dash-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dash-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: dash-admin
    namespace: kubernetes-dashboard
```

```bash
kubectl apply -f dash-admin.yaml
kubectl -n kubernetes-dashboard create token dash-admin
```

`create token` 会在终端打印一段有限期的令牌。把它粘贴到网页的令牌登录框。不要把令牌写进笔记、截图或仓库。1.24 之后 ServiceAccount 不再自动生成永久 Secret，旧的 `get secret` 管道经常是空的，优先使用 `create token`。

登录后应能看到 `default` 里的 `notice` Deployment，以及 `kube-system` 的系统 Pod。若页面能开但提示权限不足，说明绑定的命名空间或 ServiceAccount 名字和创建令牌时不一致。

实验结束后删掉管理员绑定，避免一个长期令牌留在集群里：

```bash
kubectl delete -f dash-admin.yaml
```

## init 实际做了哪些阶段

`kubeadm init` 不是一条原子操作。中途失败时，日志会停在某个阶段，重跑前要知道哪些已经写进磁盘。

1. preflight 检查 CPU、内存、swap、端口和是否已有配置。swap 没关、6443 被占用，都会在这里停下，此时还没有改 etcd。
2. 生成 CA 和各组件证书，放到 `/etc/kubernetes/pki`。重复执行 init 前如果这目录还在，要先 `kubeadm reset`，否则会沿用半套旧证书。
3. 为管理员、控制器管理器和调度器写 kubeconfig。
4. 把 apiserver、controller-manager、scheduler、etcd 的静态 Pod 清单写到 `/etc/kubernetes/manifests`。kubelet 监视这个目录，清单一出现就会把它们拉起来，不经过 Deployment。
5. 准备引导令牌和 CA 哈希，供后面的 `kubeadm join` 使用。令牌默认会过期。
6. 部署 coredns 和 kube-proxy。它们是普通的集群插件，不是静态 Pod。coredns 在 CNI 就绪前也可能起不来，这不代表 init 失败。
7. 给控制面打上 `node-role.kubernetes.io/control-plane:NoSchedule` 污点。所以验收用的 `notice` 不会出现在 cp-1 上。

`kubeadm reset` 会清掉上述清单、证书和本地 etcd 数据。只想重装网络插件时不要 reset，直接删掉对应 DaemonSet 再 apply。

## Dashboard 清单里有什么

上游的 recommended 清单很长，因为它要在一个命名空间里放齐运行网页所需的全部对象。阅读时按组看，不必逐行抄进笔记。

- 命名空间 `kubernetes-dashboard`，把管理面和业务隔开。
- ServiceAccount，以及只允许该账号读取本命名空间里若干 Secret、ConfigMap 的 Role。另外有一个 ClusterRole，允许从 metrics.k8s.io 读取 Pod 和 Node 的指标。
- 若干 Opaque Secret，用来放自签证书、CSRF 材料和密钥句柄。证书 Secret 挂到容器的 `/certs`。
- 网页 Deployment：镜像监听 8443，参数里打开自动签发证书，并用 HTTPS 探针访问根路径。根文件系统只读，临时目录用 emptyDir。节点选择 Linux，并容忍控制面污点，这样只有控制面可调度时页面仍能起来。
- 网页 Service 默认是 ClusterIP，端口 443 转到容器 8443。前面把它改成 NodePort 31443，只为实验方便。
- 另一个很小的 Deployment 负责抓取指标，自己的 Service 端口是 8000，探针走 HTTP。它和网页共用同一个 ServiceAccount。

这些对象的名字、标签都以 `k8s-app` 串起来。改 Service 类型时不要把选择器一起改掉，否则 NodePort 没有后端。

登录不用永久令牌。`kubectl create token` 签发的是有限期 JWT，过期后重新签发即可。若必须给自动化使用，再单独建 Secret 并限制它的 Role，不要把 cluster-admin 写进自动续期的流水线。

## 用一份清单把对象关系走通

命令行创建适合演示，留档时改成清单。下面在命名空间 `lab` 放一个公告页，副本数 2，Service 只在集群内暴露 8080。标签统一用 `app: notice`，三个对象靠这个键串起来。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notice
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notice
  template:
    metadata:
      labels:
        app: notice
    spec:
      containers:
        - name: httpd
          image: httpd:2.4.58-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: notice
  namespace: lab
spec:
  selector:
    app: notice
  ports:
    - name: http
      port: 8080
      targetPort: 80
```

```bash
kubectl apply -f notice.yaml
kubectl -n lab rollout status deploy/notice
kubectl -n lab get deploy,po,svc,endpointslices
```

期望看到两个 Pod 分布在 wk-1 和 wk-2（资源不够时也可能在同一台），Endpoints 里有两个地址。然后在网页里切到命名空间 `lab`，应能看到同名 Deployment。若网页里没有、命令行里有，多半是登录身份看不到该命名空间，而不是工作负载没创建。

删除时删命名空间即可，Deployment 和 Service 会一起被收走：

```bash
kubectl delete ns lab
```

## 节点状态怎么读

`kubectl get nodes` 里的 STATUS 来自一组条件，不要只看 Ready 一个词。

- Ready 为 False，且 kubelet 能连上：多半是 CNI 没把 Pod 网段铺好，先看 `kube-system` 里的 calico Pod。
- NetworkUnavailable 为 True：插件还没上报网络就绪，新 Pod 即使被调度也会停在 ContainerCreating。
- MemoryPressure 或 DiskPressure：节点在驱逐，优先清理镜像和日志，而不是继续加大副本。
- 工作节点长时间 NotReady 且 `describe` 里 Heartbeat 超过几分钟：检查该机 kubelet 是否在重启，以及到 `k8s-api.lab.local:6443` 的连通性。

控制面组件不在 `kubectl get pods` 的 default 命名空间里。静态 Pod 会出现在 `kube-system`，名字带节点名，例如 `etcd-cp-1`。看不到它们时，到 cp-1 上看 `/etc/kubernetes/manifests` 是否还在，而不是在 Deployment 列表里找 etcd。清单被挪走后，kubelet 会删掉对应的静态 Pod，控制面 API 随即不可用，工作节点上已经运行的业务容器却还会再活一段时间。

## 常见问题

- init 成功但节点一直 NotReady：几乎都是 CNI 没装、没就绪，或 Pod 网段和插件清单不一致。
- join 报证书哈希不匹配：用了别的集群的旧命令。在当前控制面重新 `kubeadm token create --print-join-command`。
- 工作节点 Ready，但新建 Pod 一直 Pending：看 `kubectl describe pod`，常见是镜像仓库不通，或节点污点把 Pod 赶走了。控制面默认带 `control-plane` 污点，业务 Pod 不会调度上去，这是正常的。
- Dashboard 使用 cluster-admin 只适合封闭实验。接到真实业务前换成只读角色，并把入口收到内网。
