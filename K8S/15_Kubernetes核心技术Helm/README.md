# Helm：把一套 YAML 当成一个应用来发布

Helm 是 Kubernetes 的包管理器，角色接近系统里的软件包工具：把分散的清单收成一个 Chart，用一条命令安装、升级或回退。

## 为什么需要它

一个服务往往同时带有 Deployment、Service，有时还有 Ingress 和配置对象。五个服务就会拆成十几份清单，散落在不同目录。恢复环境时很难判断先后，也不知道哪一份依赖另一份。

清单少的时候，直接 `kubectl apply` 完全够用。服务数量上来之后，问题集中在三处：

- 同一套清单要在测试、预发各放一份，复制粘贴后端口和副本数逐渐分叉。
- 回滚时要记住上一次每一份文件的内容，而不是一个应用版本号。
- 组件之间谁先谁后，没有一个地方统一记录。

Helm 把这些文件收进同一个 Chart：可以打包、可以复用模板、一次发布对应一个 Release。Release 才是应用级别的版本，而不是单个 Deployment 的 `revision`。

## 几个名词

- Helm：本机的命令行程序，负责创建 Chart、渲染模板、把结果交给集群。
- Chart：一份应用描述，里面是模板、默认值和元数据，还不是集群里的对象。
- Release：某次把 Chart 安装到集群后的实例。同一个 Chart 可以装出多个 Release，各自有名字和历史。
- Repository：存放打包后 Chart 的仓库，客户端用 HTTP 拉取索引和压缩包。

## 组件怎么演进

Helm 2 分成客户端和集群内的 Tiller。Tiller 持有较高权限，负责把渲染结果写入 apiserver，Chart 仓库只被客户端访问。

2019-11-13 发布的 Helm 3 去掉了 Tiller。客户端拿着 kubeconfig 直接和 apiserver 通信，权限跟当前用户一致，不再有一个常驻集群的高权限组件。Release 的存储也改到了它所在的命名空间，因此同名 Release 可以在不同命名空间各装一份。Helm 2 里 Release 名在整个集群只能有一个。

现在新环境只需要安装 Helm 3 客户端，不必再部署 Tiller，也不必给 Tiller 单独做 RBAC。

## 安装客户端与仓库

从 Helm 的发行页下载与系统匹配的压缩包，解压后把二进制放到 `PATH` 里，例如 `/usr/local/bin/helm`，然后执行 `helm version` 确认是 v3。

仓库需要自己添加。下面用公开仓库做演示，别名不要和别人机器上的历史别名混用：

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable-mirror https://mirror.azure.cn/kubernetes/charts
helm repo update
helm repo list
helm search repo bitnami/nginx
```

不需要的源可以删掉：

```bash
helm repo remove stable-mirror
```

`search repo` 只查已经添加的仓库。刚加完源却搜不到包，先执行一次 `helm repo update` 刷新本地索引。

## 三条生命周期命令

- `helm install`：第一次把 Chart 装成 Release
- `helm upgrade`：用新的模板或参数替换现有 Release，并留下修订号
- `helm rollback`：按修订号退回以前的渲染结果

## 从仓库装一个应用

搜索并安装。这里的 Release 名叫 `edge-ui`，Chart 选择仓库里的 nginx，而不是把别的可视化组件硬装进来：

```bash
helm search repo bitnami/nginx
helm install edge-ui bitnami/nginx --namespace edge --create-namespace
helm list -n edge
helm status edge-ui -n edge
```

`status` 会打印本次生成的对象，以及 Chart 作者写的访问说明。若 Service 仍是 ClusterIP，集群外没有入口。可以用补丁改成 NodePort，而不是手改控制器已经接管的对象后被下一次升级冲掉：

```bash
kubectl -n edge patch svc edge-ui-nginx \
  -p '{"spec":{"type":"NodePort","ports":[{"port":80,"nodePort":31880}]}}'
kubectl -n edge get svc edge-ui-nginx
```

然后用“节点 IP:31880”访问。更稳妥的做法是安装时传入 Chart 自己的 values，让 Service 类型从一开始就是 NodePort，升级时也不会被默认值盖回去。

想先看渲染结果再决定是否提交：

```bash
helm template edge-ui bitnami/nginx
```

## 自己写一个 Chart

```bash
helm create shopchart
```

当前目录会出现 `shopchart/`。需要记住的三块：

- `templates/`：Deployment、Service 等模板
- `values.yaml`：模板里引用的默认参数
- `Chart.yaml`：Chart 名称、版本、描述

可以先用 kubectl 导出一份骨架，再改名、改镜像，放进 `templates/`。注意加上 `--dry-run=client`，避免在集群里先造出一个同名对象：

```bash
kubectl create deployment storefront --image=httpd:2.4-alpine \
  --dry-run=client -o yaml > shopchart/templates/deployment.yaml
kubectl expose deployment storefront --port=8080 --target-port=80 \
  --type=NodePort --dry-run=client -o yaml > shopchart/templates/service.yaml
```

`expose` 在对象还不存在时，必须依赖 `--dry-run=client` 才能只生成清单。导出的文件里常带 `creationTimestamp: null` 和空的 `status`，放进模板前删掉，否则升级时会和真实对象的状态打架。

安装这份本地 Chart：

```bash
helm install storefront ./shopchart --namespace edge
```

改完模板或 values 之后：

```bash
helm upgrade storefront ./shopchart --namespace edge
helm history storefront -n edge
```

## 用 values 渲染模板

把会变的字段留在 `values.yaml`，模板只保留结构。一份商店前端可能变化的是镜像、标签、副本数和端口：

```yaml
replicaCount: 2
image:
  repository: httpd
  tag: "2.4.58"
service:
  type: NodePort
  port: 8080
  nodePort: 31881
```

模板里用 `{{ .Values.镜像字段 }}` 取值，Release 名字用 `{{ .Release.Name }}`，这样同一份 Chart 装两次不会撞名：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "{{ .Release.Name }}"
spec:
  replicas: "{{ .Values.replicaCount }}"
  selector:
    matchLabels:
      app: "{{ .Release.Name }}"
  template:
    metadata:
      labels:
        app: "{{ .Release.Name }}"
    spec:
      containers:
        - name: httpd
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
```

提交前先渲染，确认没有空值：

```bash
helm install --dry-run storefront ./shopchart -n edge
```

输出里应能看到镜像 `httpd:2.4.58`、副本数 2，以及名字 `storefront`。确认无误后再去掉 `--dry-run`。

## 常见问题

- 升级时如果安装阶段用 `kubectl patch` 改过的字段没有写进 values，下一次 `helm upgrade` 会按 Chart 默认值把改动盖掉。
- 模板渲染出的名字超过 63 个字符，或者带有大写字母，apiserver 会拒绝 Service。
- `--dry-run` 在 Helm 3 里默认只在客户端渲染；要连集群校验，使用 `--dry-run=server`。
- 旧教程里的 `stable` 仓库已经停止维护，搜不到包时先换仍在更新的仓库，而不是反复执行 update。
