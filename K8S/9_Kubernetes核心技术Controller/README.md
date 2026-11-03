# 工作负载 Controller

## 本章内容

- 控制器在集群里做什么
- 它和 Pod 如何对应
- Deployment 适合哪些服务
- 清单里和发布相关的字段
- 用清单创建并暴露
- 更换镜像与回退
- 改副本数

## 什么是控制器

控制器是一直运行的纠偏循环：读取某类对象的期望状态，再把集群调到接近它。Pod 是被管的运行单元，本身不会在失败后按模板再生。说“控制器存在、Pod 可替换”指的就是这层关系。

## 和 Pod 的关系

扩容、滚动替换、失败补齐都由控制器完成。它靠标签选择器认领 Pod，而不是记下创建时的随机后缀。选择器写错时，控制器会不停创建新 Pod，却认为已有的那些不属于自己。

## Deployment 适合什么

- 副本之间可以互换的服务
- 它管理 ReplicaSet，ReplicaSet 再管理 Pod
- 自带滚动发布和修订记录
- 典型对象是网站、网关和多数无状态微服务

一次发布往往不是改某个 Pod，而是换一套模板：新 ReplicaSet 的副本逐渐增加，旧 ReplicaSet 逐渐减到 0。单个 ReplicaSet 描述不了“两代模板同时存在”这件事，所以外面还要有 Deployment。长期在线、不需要稳定编号的业务，优先用它。

## 用 Deployment 发布

命令行能直接创建：

```bash
kubectl -n shop create deployment notice --image=nginx:1.25.4
```

这条命令不好评审，也不能重复执行出同一份规格。先导出骨架：

```bash
kubectl -n shop create deployment notice --image=nginx:1.25.4 \
  --dry-run=client -o yaml > notice.yaml
```

骨架里和关联有关的部分如下。`metadata.labels`、`selector.matchLabels` 和模板标签三处的 `app: notice` 必须一致。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notice
  namespace: shop
  labels:
    app: notice
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
        - name: notice
          image: nginx:1.25.4
```

### 提交并用 Service 暴露

```bash
kubectl apply -f notice.yaml
kubectl -n shop expose deployment notice \
  --name=notice-np --port=80 --target-port=80 --type=NodePort
```

参数含义：

- `--port`：Service 自己的端口，集群内访问用它
- `--target-port`：容器里进程监听的端口
- `--name`：Service 对象的名字，默认为 Deployment 同名
- `--type`：`NodePort` 会在每个节点上再开一个高位端口

Service 也可以先导出再改，例如固定 `nodePort: 31880`：

```bash
kubectl -n shop expose deployment notice \
  --name=notice-np --port=80 --target-port=80 --type=NodePort \
  --dry-run=client -o yaml > notice-np.yaml
```

一份去掉状态字段后的 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: notice-np
  namespace: shop
  labels:
    app: notice
spec:
  type: NodePort
  selector:
    app: notice
  ports:
    - port: 80
      targetPort: 80
      nodePort: 31880
      protocol: TCP
```

```bash
kubectl apply -f notice-np.yaml
kubectl -n shop get pods,svc -l app=notice
```

浏览器访问 `http://10.20.8.21:31880/` 应看到默认欢迎页。若连接失败，先确认 Pod 是 Running，再确认安全组放行了 31880。

## 升级、回退和伸缩

- 升级：模板里的镜像从 `1.25.4` 换成 `1.25.5`，旧副本在新副本就绪前仍接流量
- 回退：把模板指回上一修订
- 伸缩：只改副本数，镜像不变

### 更换镜像

清单里先固定一个明确标签：

```yaml
image: nginx:1.25.4
```

```bash
kubectl apply -f notice.yaml
kubectl -n shop set image deployment/notice notice=nginx:1.25.5
kubectl -n shop rollout status deployment/notice
```

滚动过程中能看到两代 Pod 短暂并存：新 Pod 进入 Running 且就绪后，旧 Pod 才被删掉。若新镜像一直拉不下来，发布会停住，旧副本继续服务。这就是“先增后减”，避免把唯一的副本先杀掉。

在节点上执行 `docker images` 或 `crictl images`，应能看到新拉下来的 `nginx:1.25.5`。

#### 看进度和修订

```bash
kubectl -n shop rollout status deployment/notice
kubectl -n shop rollout history deployment/notice
```

历史里的修订号对应一次模板变更。只改副本数有时不会单独形成你期望的业务版本，以 `CHANGE-CAUSE` 和镜像字段为准。

#### 回退

回到上一修订：

```bash
kubectl -n shop rollout undo deployment/notice
kubectl -n shop rollout status deployment/notice
```

回到指定修订，例如 2：

```bash
kubectl -n shop rollout undo deployment/notice --to-revision=2
```

回退同样是一次滚动，不是把进程在原地降级。完成后用 `kubectl -n shop get pods -o jsonpath='{.items[*].spec.containers[*].image}'` 核对镜像标签。

### 伸缩

```bash
kubectl -n shop scale deployment/notice --replicas=6
kubectl -n shop get pods -l app=notice
```

副本从 2 改到 6 时，ReplicaSet 补 4 个 Pod，调度器再把它们铺到有余量的节点。缩回 2 时删掉多余的，Service 的端点跟着变。需要按 CPU 自动调整时再使用 HPA，那是另一类控制器，不替代这里的手动改数。
