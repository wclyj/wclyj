# 用 kubeadm-ha 一次搭起一套集群

## 为什么改走脚本

按官方步骤手工装时，控制面镜像默认从境外仓库拉取，实验室网络里经常卡在 `ImagePullBackOff`，重试几次后 `kubeadm init` 直接失败。社区项目 [kubeadm-ha](https://github.com/TimeBye/kubeadm-ha) 用 Ansible 把源、内核参数、运行时和 kubeadm 收成剧本，适合“先得到一套能用的 1 主多从，再回头看每一步”。下面补的是仓库 README 里一笔带过、真正卡住人的地方。

## 机器与系统

控制面和工作节点都按 2 核 4G 起配。演示用三台系统，一台控制面、两台工作节点，etcd 三台都参加，满足奇数。

| 主机名 | 地址 | 规格 |
| --- | --- | --- |
| ctrl-a | 10.8.20.11 | 2C4G |
| worker-b | 10.8.20.12 | 2C4G |
| worker-c | 10.8.20.13 | 2C4G |

示例系统为 CentOS 7 系。生产环境更宜换成仍在维护的兼容发行版，剧本若只声明了 el7，要先在仓库里确认分支是否支持新系统。

### 网络与软件

三台都要能出网，否则 Ansible 拉不到包：

```bash
ping -c 2 mirrors.aliyun.com
```

实验环境关闭 firewalld，避免节点间 6443、2379 被挡。生产不要无条件关闭，应改成按端口放行。

```bash
systemctl disable --now firewalld
systemctl status firewalld
```

再确认三台互访，例如在 ctrl-a 上：

```bash
ping -c 2 10.8.20.12
ping -c 2 10.8.20.13
```

只在控制面安装 Git，用来取剧本：

```bash
yum install -y git
```

## 部署前的配置

### 取得剧本

```bash
git clone --depth 1 https://github.com/TimeBye/kubeadm-ha
cd kubeadm-ha
sudo ./install-ansible.sh
```

`install-ansible.sh` 会在控制面装上 Ansible。失败时先看是不是 Python 或 epel 源不通，不要直接跑后面的 playbook。

### 编写主机清单

单控制面使用仓库自带的 `example/hosts.s-master.ip.ini`，改地址和登录口令。下面是一份改写后的示例，口令仅用于隔离的实验虚拟机：

```ini
; 字段依次是：内网地址、sshd 端口、登录用户、登录口令
[all]
10.8.20.11 ansible_port=22 ansible_user="root" ansible_ssh_pass="Lab#2026"
10.8.20.12 ansible_port=22 ansible_user="root" ansible_ssh_pass="Lab#2026"
10.8.20.13 ansible_port=22 ansible_user="root" ansible_ssh_pass="Lab#2026"

; 单控制面不需要独立的负载均衡组
[lb]

; etcd 成员数保持奇数
[etcd]
10.8.20.11
10.8.20.12
10.8.20.13

[kube-master]
10.8.20.11

[kube-worker]
10.8.20.11
10.8.20.12
10.8.20.13

[new-master]
[new-worker]
[new-etcd]
[del-worker]
[del-master]
[del-etcd]
[del-node]

[all:vars]
skip_verify_node=true
kube_version="1.26.15"
lb_mode="nginx"
lb_kube_apiserver_port="8443"
kube_pod_subnet="10.88.0.0/18"
kube_service_subnet="10.88.64.0/18"
kube_network_node_prefix="24"
kube_max_pods="110"
network_plugin="calico"
kubelet_root_dir="/var/lib/kubelet"
docker_storage_dir="/var/lib/docker"
etcd_data_dir="/var/lib/etcd"
```

这组 Pod 网段 `10.88.0.0/18` 与 Service 网段 `10.88.64.0/18` 故意避开服务器所在的 `10.8.20.0/24`。若机房已经占用 `10.88`，换成另一段不重叠的地址，否则跨节点通信会和物理路由冲突。`/18` 大约 1.6 万个 Pod 地址，节点前缀 `/24` 时大约能容纳 64 台节点。

`skip_verify_node=true` 会跳过规格检查。演示机内存偏小时可以用，生产应改回严格检查，避免 kubelet 在内存压力下被系统杀掉。

控制面同时也写进了 `[kube-worker]`，表示这台机器要跑业务 Pod。资源紧的实验可以这样；生产通常让控制面打上不可调度污点，业务只落在 worker-b 和 worker-c。

### 升级内核并重启

```bash
ansible-playbook -i example/hosts.s-master.ip.ini 00-kernel.yml
```

剧本成功后，三台分别执行 `reboot`。等 SSH 恢复再继续，不要在重启过程中启动初始化。

## 初始化集群

```bash
cd kubeadm-ha
ansible-playbook -i example/hosts.s-master.ip.ini 90-init-cluster.yml
kubectl get nodes
```

三台都变成 Ready 即算完成。角色列里应能看到 ctrl-a 带 master（以及可能的 etcd、worker），另外两台带 worker。版本号与清单里的 `kube_version` 一致，例如 `v1.26.15`。长时间 NotReady 时，先看 CNI Pod 是否起来：`kubectl get pods -n kube-system`。

初始化失败、想连数据一起清空时：

```bash
ansible-playbook -i example/hosts.s-master.ip.ini 99-reset-cluster.yml
```

重置会删掉 etcd 数据和工作负载，只在确认没有需要保留的状态时执行。

## 可选：先准备容器运行时的镜像加速

kubeadm-ha 通常已经装好运行时。若还要在控制面上单独补 Docker，以便手工拉镜像，可以换国内软件源后再装：

```bash
cat >/etc/yum.repos.d/docker-ce.repo <<'EOF'
[docker-ce-stable]
name=Docker CE Stable
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
EOF

yum -y install docker-ce
systemctl enable --now docker
```

镜像加速写到 daemon 配置里。加速地址用你自己的仓库分配的域名，不要复用别人账号下的专属域名：

```bash
mkdir -p /etc/docker
cat >/etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": ["https://docker.m.daocloud.io"]
}
EOF
systemctl restart docker
```

## 可选：图形界面 Kuboard

[Kuboard](https://kuboard.cn/) 提供一套基于浏览器的集群视图，适合不想记全部 kubectl 子命令的时候做巡检。它和后面的 Rancher 二选一即可，两个都装会多占控制面内存。

在能访问 apiserver 的机器上应用上游清单：

```bash
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.7/metrics-server.yaml
kubectl get pods -n kube-system -l k8s.kuboard.cn/name=kuboard
```

Pod 从 ContainerCreating 变成 Running 之后再打开页面。入口是 NodePort，具体端口以 Service 为准，不要凭记忆写死：

```bash
kubectl get svc -n kube-system -l k8s.kuboard.cn/name=kuboard
```

假设输出的节点端口是 `31240`，浏览器访问 `http://10.8.20.12:31240/`。首次登录要填 ServiceAccount 令牌，在控制面执行下面的管道取得，把它粘贴到登录框，不要把令牌写进笔记或截图：

```bash
kubectl -n kube-system get secret \
  "$(kubectl -n kube-system get secret | awk '/kuboard-user/{print $1; exit}')" \
  -o go-template='{{.data.token}}' | base64 -d
echo
```

较新的 Kubernetes 不再自动为 ServiceAccount 创建长期 Secret。若上面的命令找不到 Secret，需要按当前版本文档创建令牌，或改用 `kubectl create token`。

不再使用时按同样的地址删除：

```bash
kubectl delete -f https://kuboard.cn/install-script/kuboard.yaml
kubectl delete -f https://addons.kuboard.cn/metrics-server/0.3.7/metrics-server.yaml
```

## 可选：用 Helm 安装 Rancher

与 Kuboard 二选一。先放一个确定版本的 Helm 3 客户端，避免使用发行版里过旧的 v2：

```bash
curl -fL -o /tmp/helm.tgz \
  https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz
tar -zxvf /tmp/helm.tgz -C /tmp
install -m 0755 /tmp/linux-amd64/helm /usr/local/bin/helm
helm version
```

版本行里应出现 `v3.14.4`。然后添加 Rancher 的稳定仓库并安装到专用命名空间。主机名用实验域名 `ranch.lab.internal`，客户端 hosts 要能解析到入口地址：

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm install rancher rancher-stable/rancher \
  --create-namespace \
  --namespace cattle-system \
  --set hostname=ranch.lab.internal \
  --set replicas=1
kubectl -n cattle-system rollout status deploy/rancher
```

副本数在实验环境收成 1，避免三副本把 4G 内存打满。看到 `successfully rolled out` 后，按 Chart 说明取出引导口令。若一直停在 `0 of 1 updated replicas are available`，先看是不是 Ingress 控制器还没装，Rancher 的入口依赖它。

## 常见问题

- 清单里的 Pod 网段和机房网段重叠时，跨节点 Pod 能解析域名但 ping 不通别的节点上的 Pod。
- Ansible 使用口令登录只适合实验。剧本能跑通之后应换成 SSH 密钥，并把含口令的 ini 排除出版本库。
- 重置剧本不可逆。需要保留 etcd 数据时先做快照，再决定是否执行 `99-reset-cluster.yml`。
- 图形界面和业务负载抢控制面内存时，优先把界面副本挪到工作节点，或在实验结束后删掉。
