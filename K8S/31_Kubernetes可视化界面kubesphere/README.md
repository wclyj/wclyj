# KubeSphere 控制台

## 前言

Kubernetes 自带的 Dashboard 能看工作负载，但流水线、多租户和可插拔组件要另搭。KubeSphere 在集群之上提供这些控制台能力，组件多，机器也要更大。若只想要一个轻量的对象查看页，可以评估 Kuboard 这类项目，它的资源占用通常更低。

## 简介

KubeSphere 是面向多团队的开源管理平面：底层仍是 Kubernetes，界面把常见操作收成向导，用来降低日常发布和排障时直接写清单的频率。调度、Service 和存储语义没有另起一套，控制台只是 API 的客户端。

## 安装

### 前提

以 3.3 这一代的最低要求为例，安装前核对：

- 集群版本落在该发行说明支持的范围内（常见为 1.19 到 1.22 这一段，以你选的安装包说明为准）
- 安装机至少 2 核 CPU、4 GB 内存；打开 DevOps、日志等组件后要按说明继续加
- 集群里已经有默认 StorageClass，安装器要创建持久卷
- API 进程开启了 CSR 签名相关参数，节点证书申请才能通过
- 更细的端口、权限和磁盘检查以该版本的安装前提清单为准

### 安装 Helm

Helm 是 Kubernetes 的包管理器，角色类似发行版里的 yum 或 Python 里的 pip：按图表查找、安装和升级一组对象。3.x 起客户端直接驱动安装，不再依赖集群里的 Tiller。

在能访问 API 的控制面节点上：

1. 从 Helm 的发布页取得对应系统的压缩包，例如 `helm-v3.12.3-linux-amd64.tar.gz`
2. 解压后把其中的 `helm` 二进制放到 `/usr/local/bin`
3. 执行 `helm version`，应只打印客户端版本

本章的 KubeSphere 仍用官方安装器清单，不强制改成 Helm 仓库安装。Helm 留着给以后装 Ingress 或监控图表。

## 部署

### 安装前检查

默认存储类不存在时，安装器会卡在等待卷绑定。先确认：

```bash
kubectl get storageclass
```

输出里某一行带 `(default)` 才能继续。没有的话先装 NFS 或本地卷供给器，并把它标成默认类。NFS 服务端若已装好但没启动：

```bash
systemctl enable --now nfs-server
```

安装机器如果拉不到发布页，就把下面两个文件提前下载到本机，文件名保持不变。

### 提交安装器

在线安装使用与所选版本一致的两份清单。下面以 v3.3.2 为例：

```bash
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.2/kubesphere-installer.yaml
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.2/cluster-configuration.yaml
```

已经落在当前目录时：

```bash
kubectl apply -f kubesphere-installer.yaml
kubectl apply -f cluster-configuration.yaml
```

要卸掉控制台而保留业务集群时，对这两份文件执行 `kubectl delete -f`。不要用删除命名空间的方式硬清，卷和 CRD 容易留下。

### 看安装日志

安装器 Pod 会按 `cluster-configuration` 里打开的组件逐个拉起。跟踪日志：

```bash
kubectl logs -n kubesphere-system \
  -l app=ks-install -f
```

另开一个终端看该空间的 Pod：

```bash
kubectl get pod -n kubesphere-system
```

刚开始会有若干 Pod 停在 ContainerCreating 或 Init。再用全集群视角确认相关命名空间没有持续 CrashLoopBackOff：

```bash
kubectl get pods -A | grep -E 'kubesphere|kubesphere-system'
```

控制台服务起来后查看端口：

```bash
kubectl get svc ks-console -n kubesphere-system
```

类型一般是 NodePort，端口 30880。安全组或防火墙要放行这个端口。浏览器访问 `http://10.20.8.11:30880/`。

初始账号口令是安装器写入的 `admin` / `P@88w0rd`。登录后立刻改掉。组件页里若有项仍在启动，对应菜单会暂时不可用，等 Pod Ready 后再打开，不必重复安装。

## 错误排查

### 提示 account is not active

默认用户要等 `ks-controller-manager` 就绪后，才会被用户控制器加密口令并标成可登录。若页面报 account is not active，先看这个 Deployment 是否在重启，以及它的日志里有没有 webhook 证书错误。

证书和校验 webhook 不一致时，可以把两者导出来核对 `caBundle`，把控制器使用的 CA 写回校验配置，再滚动重启：

```bash
kubectl -n kubesphere-system get validatingwebhookconfiguration users.iam.kubesphere.io -o yaml > users-webhook.yaml
kubectl -n kubesphere-system get secret ks-controller-manager-webhook-cert -o yaml > ks-webhook-cert.yaml
kubectl -n kubesphere-system apply -f ks-webhook-cert.yaml
kubectl apply -f users-webhook.yaml
kubectl -n kubesphere-system rollout restart deploy ks-controller-manager
```

重启完成、Pod Ready 之后再用初始口令登录。仍失败就继续看控制器日志，而不是反复重置口令。
