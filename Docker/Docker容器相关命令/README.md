# 容器相关命令备忘

命令在 Ubuntu 22.04 上整理，口令请自换。

涉及的平台：

- Rancher
- Kubernetes
- Helm
- Docker

上面要跑的服务：

- Prometheus
- Grafana
- PostgreSQL

# 1. Docker

## 1.1. apt 源

先备份，再换成国内镜像。

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.orig

cat > /etc/apt/sources.list << 'EOF'
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
EOF

apt-get update
```

`deb` 是二进制包。`>` 覆盖文件，`>>` 才追加。

## 1.2. 安装 Docker

步骤对应官方的 Ubuntu 安装说明：https://docs.docker.com/engine/install/ubuntu/

```bash
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

锁定版本时先查再装：

```bash
apt-cache madison docker-ce | head
sudo apt-get install -y docker-ce=5:26.1.4-1~ubuntu.22.04~jammy docker-ce-cli=5:26.1.4-1~ubuntu.22.04~jammy containerd.io
```

版本串要和查询结果一致。

## 1.3. 镜像加速

镜像站地址换成自己的。没有证书的私仓才放进 `insecure-registries`。

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null << 'EOF'
{
  "registry-mirrors": ["https://docker.m.daocloud.io"],
  "insecure-registries": ["10.8.20.40:5000"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 1.4. PostgreSQL

镜像说明：https://hub.docker.com/_/postgres

```bash
docker run -d \
  --name ledger-db \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD='lab-pg-6612' \
  -e POSTGRES_DB=ledger \
  postgres:16
```

`-e` 传环境变量，`-it` 打开终端。`--rm` 不要和 `-d` 一起用。

```bash
docker exec -it ledger-db bash
psql -U postgres -d ledger
```

```text
ledger=# create table account (id int primary key, name text not null);
CREATE TABLE
ledger=# insert into account values (7, 'north');
INSERT 0 1
ledger=# select id, name from account;
 id | name
----+-------
  7 | north
(1 row)
ledger=# \q
```

`\q` 之后再 `exit`。

### 1.4.1. pgagent

官方镜像没有 pgAgent，补一层再构建：

```bash
mkdir -p /opt/build/pgagent
cat > /opt/build/pgagent/Dockerfile << 'EOF'
FROM postgres:16
RUN apt-get update \
 && apt-get install -y --no-install-recommends pgagent \
 && rm -rf /var/lib/apt/lists/*
EOF
docker build -t ledger-pgagent:16 /opt/build/pgagent
```

启动并装上扩展：

```bash
docker run -d --name ledger-agent \
  -e POSTGRES_PASSWORD='lab-pg-6612' \
  -p 5433:5432 \
  ledger-pgagent:16

docker exec -it ledger-agent bash
psql -U postgres -c 'CREATE EXTENSION pgagent;'
pgagent hostaddr=127.0.0.1 dbname=postgres user=postgres password='lab-pg-6612' -s /tmp/pgagent.log
exit
```

连接参数在 pgAdmin 里填。日志为空就先看进程。

### 1.4.2. pgAdmin

```bash
docker run -d --name ledger-admin \
  -p 15050:80 \
  -e PGADMIN_DEFAULT_EMAIL=ops@example.net \
  -e PGADMIN_DEFAULT_PASSWORD='lab-admin-4410' \
  dpage/pgadmin4:8
```

打开 `http://主机:15050` 登录。不在同一网络时不要填 `127.0.0.1`。

## 1.5. Prometheus

采集目标按 `job_name` 分组。

```bash
mkdir -p /opt/obs
vi /opt/obs/prom.yml

docker run -d --name obs-prom \
  -p 19090:9090 \
  -v /opt/obs/prom.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus:v2.53.0 \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

`-p`、`-v` 都是左边宿主机、右边容器。

`/opt/obs/prom.yml` 示例：

```yaml
scrape_configs:
  - job_name: obs-prom
    static_configs:
      - targets: ['10.8.20.21:19090']
  - job_name: pg-export
    static_configs:
      - targets: ['10.8.20.21:19187']
  - job_name: node-export
    static_configs:
      - targets: ['10.8.20.21:19100']
```

改完配置不会自动生效。加了 `--web.enable-lifecycle` 后可以重载：

```bash
curl -X POST http://10.8.20.21:19090/-/reload
```

也可对进程发 `SIGHUP`。失败则仍用旧配置。

### 1.5.1. postgres_exporter

项目页：https://github.com/prometheus-community/postgres_exporter

```bash
docker run -d --name pg-export \
  -p 19187:9187 \
  -e DATA_SOURCE_NAME='postgresql://postgres:lab-pg-6612@10.8.20.21:5432/ledger?sslmode=disable' \
  quay.io/prometheuscommunity/postgres-exporter:v0.15.0
```

连接串里的口令能被 `inspect` 看到。

### 1.5.2. node_exporter

```bash
docker run -d --name node-export \
  --net host \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/host/rootfs:ro \
  prom/node-exporter:v1.8.1 \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys \
  --path.rootfs=/host/rootfs
```

`--net host` 后进程直接听宿主机 9100。

## 1.6. Grafana

```bash
docker run -d --name obs-board \
  -p 13000:3000 \
  grafana/grafana:11.1.0
```

初始账号口令都是 `admin`。数据源填 `http://10.8.20.21:19090`。

### 1.6.1. 钉钉报警

群机器人要设关键词。Grafana 联系点贴上 webhook，再用测试按钮确认。

### 1.6.2. 仪表盘模板

面板从 https://grafana.com/grafana/dashboards/ 按编号导入。全是 No data 时，先确认采集目标为 UP。

# 2. Rancher

单节点安装见官方文档。标签写死版本，不用 `:stable`。

```bash
docker run -d --name ranch-lab \
  --restart unless-stopped \
  -p 18080:80 -p 18443:443 \
  rancher/rancher:v2.8.5
```

拉取慢时改用阿里云上的同版本镜像：

```bash
docker run -d --name ranch-lab \
  --restart unless-stopped \
  -p 18080:80 -p 18443:443 \
  registry.cn-hangzhou.aliyuncs.com/rancher/rancher:v2.8.5
```

打开 `https://主机:18443`，把注册命令拿到节点上执行。

# 3. Kubernetes

下面用 kubeadm 在三台 Ubuntu 上搭 1.28 练习集群。

## 3.1. 准备环境

改 SSH 端口后：

```bash
systemctl restart ssh
```

主机名：

```bash
hostnamectl set-hostname ctrl-a
```

`/etc/hosts` 三台都要有：

```text
10.8.20.11 ctrl-a
10.8.20.12 work-b
10.8.20.13 work-c
```

分发：

```bash
scp -P 22 /etc/hosts root@work-b:/etc/hosts
```

各节点 `swapoff -a`，并注释 fstab 里的 swap。

## 3.2. Docker

安装见 1.2。若用 Docker 当运行时，kubelet 的套接字要指向它。

## 3.3. 工具

工具包走阿里云的 Kubernetes 源：

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/k8s.gpg

echo 'deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main' \
  | sudo tee /etc/apt/sources.list.d/k8s.list

sudo apt-get update
sudo apt-get install -y kubelet=1.28.9-00 kubeadm=1.28.9-00 kubectl=1.28.9-00
sudo apt-mark hold kubelet kubeadm kubectl
```

`hold` 避免以后单独升版本。

## 3.4. 初始化集群

在 `ctrl-a` 上：

```bash
kubeadm init \
  --apiserver-advertise-address 10.8.20.11 \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version v1.28.9 \
  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
```

把管理员配置放到当前用户：

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

这个网段要和 Flannel 默认的 `10.244.0.0/16` 一致。

## 3.5. Flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

镜像慢就按 4.2 导出再拷。节点变为 Ready 即插件已生效。

## 3.6. 加入集群

在工作节点执行加入命令。令牌过期时重新打印：

```bash
kubeadm token create --print-join-command
```

示意令牌，不要照抄：

```bash
kubeadm join 10.8.20.11:6443 \
  --token a9c2e1.sampletoken0001 \
  --discovery-token-ca-cert-hash sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

## 3.7. Dashboard

下面是临时打开的 NodePort。正式环境改用 kubectl 代理。

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}'
kubectl -n kubernetes-dashboard create serviceaccount dash-admin
kubectl create clusterrolebinding dash-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dash-admin
kubectl -n kubernetes-dashboard create token dash-admin
kubectl -n kubernetes-dashboard get svc
```

令牌贴到登录框，端口看 `get svc`。要自签证书时：

```bash
mkdir -p ~/dash-certs && cd ~/dash-certs
openssl genrsa -out ca.key 2048
openssl req -x509 -new -key ca.key -out ca.crt -days 825 -subj "/CN=lab-ca"
openssl genrsa -out dash.key 2048
openssl req -new -key dash.key -out dash.csr -subj "/CN=dashboard"
openssl x509 -req -in dash.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dash.crt -days 825
```

按官方清单把证书做成 Secret。删除整个界面：

```bash
kubectl delete namespace kubernetes-dashboard
```

命名空间删掉后，里面的 Secret 一并消失。

## 3.8. Helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

不要再加已经废弃的 `stable` 仓库。

## 3.9. Stolon

安装前要有能绑定的 StorageClass，否则卷会 Pending。`values-lab.yaml`：

```yaml
image:
  repository: sorintlab/stolon
  tag: v0.17.0-pg14
persistence:
  enabled: true
  storageClassName: local-disk
  accessModes:
    - ReadWriteOnce
  size: 20Gi
superuserUsername: stolon
superuserPassword: lab-stolon-2208
replicationUsername: repl
replicationPassword: lab-repl-2208
```

```bash
helm repo add stolon https://sorintlab.github.io/stolon-chart 2>/dev/null || true
helm install lab-pg ./stolon -n pg-ha --create-namespace -f values-lab.yaml
```

想先看清单、不真正安装：

```bash
helm install lab-pg ./stolon -n pg-ha --dry-run > /tmp/stolon-render.yaml
```

手工准备本地卷（路径必须已存在）：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-disk
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pg-pv-b
spec:
  capacity:
    storage: 20Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-disk
  local:
    path: /srv/pgdata
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: ["work-b"]
```

没做初始化时，进 Keeper 执行一次，集群名与 ConfigMap 对齐：

```bash
kubectl -n pg-ha exec -it lab-pg-keeper-0 -- \
  stolonctl --cluster-name=lab-pg --store-backend=kubernetes --kube-resource-kind=configmap init
```

初始化会重写口令。只改库、不改 Secret，Proxy 会连不上。

## 3.10. 在别的机器上管理集群

运维机只装 kubectl，并拷入 `admin.conf`。

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
echo 'deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main' \
  | sudo tee /etc/apt/sources.list.d/k8s.list
sudo apt-get update && sudo apt-get install -y kubectl
```

Windows 用官方 `kubectl.exe`，同样只依赖这份 kubeconfig。

# 4. 常用命令

## 4.1. Linux

查看监听端口：

```bash
ss -lntup
```

切到性能模式：

```bash
sudo apt-get install -y linux-tools-common linux-tools-generic
sudo cpupower frequency-set -g performance
cpupower frequency-info | head
```

改端口后要重启 sshd，并同步 `scp -P`。

## 4.2. Docker

进入容器：

```bash
docker exec -it ledger-db bash
```

看完整启动命令：

```bash
docker ps -a --no-trunc
```

停掉并删除全部容器：

```bash
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
```

删除全部镜像：

```bash
docker rmi $(docker image ls -q)
```

导出再导入，用来拷 Flannel 镜像：

```bash
docker save -o /tmp/flannel.tar ghcr.io/flannel-io/flannel:v0.25.1
docker load -i /tmp/flannel.tar
```

引擎重启后仍自动拉起：

```bash
docker update --restart=unless-stopped ledger-db
```

## 4.3. Kubernetes

扩容时带上当前副本数，避免扩错：

```bash
kubectl -n pg-ha scale statefulset lab-pg-keeper --current-replicas=2 --replicas=3
kubectl -n pg-ha scale deployment lab-pg-proxy --current-replicas=2 --replicas=3
```

把服务改成 NodePort：

```bash
kubectl -n pg-ha patch svc lab-pg-proxy -p '{"spec":{"type":"NodePort"}}'
```

或在创建时暴露：

```bash
kubectl -n pg-ha expose deployment lab-pg-proxy --type=NodePort --port=5432
```

PV 停在 Released 时，删掉 `claimRef` 才能回到 Available：

```bash
kubectl edit pv pg-pv-b
```

删掉 `spec.claimRef` 再保存。卷里还有要留的数据时不要做。

# Stolon 补充

若安装清单里已有 init，可不再手工初始化。用状态确认：

```bash
kubectl -n pg-ha get pod
kubectl -n pg-ha exec lab-pg-keeper-0 -- stolonctl --cluster-name=lab-pg --store-backend=kubernetes status
```

有 master 才算复制建立。没有 master 就还没初始化，或卷没绑上。
