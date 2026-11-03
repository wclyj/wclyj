# 核心对象 Pod

## Pod 概述

Kubernetes 不直接调度单个容器。用户提交的最小对象是 Pod：一个或多个容器的组合，其它对象都在它外面帮忙。控制器负责维持 Pod 的个数，Service 和 Ingress 负责把流量引进来，PersistentVolume 负责给它盘。容器运行时只在 Pod 已经绑定到节点之后才被调用。

每个 Pod 里还有一个基础容器，镜像由平台提供，常被叫做 pause。它先占住网络命名空间，业务容器再加入这个命名空间。所以“一个 Pod”在运行时至少是 pause 加上你写的容器。

### 基本概念

- 调度、扩缩和健康检查都落在 Pod 这一层
- 里面是一组要一起放、一起停的容器
- 它们共享网络命名空间，也共享同一个 Pod IP
- Pod 本身是临时的，被驱逐或替换后 IP 不保留
- 业务容器应紧密相关，不要把互不相干的进程硬塞进同一个 Pod

### 为什么不是一个容器一个 Pod

用 Docker 时，一个容器通常就是一个进程。Pod 允许把必须放在一起的进程编成一组：它们要频繁对话，用 localhost 或本地套接字就够了，还要共享一份文件。

这种关系叫协同定位。例如页面进程和把日志尾随出去的边车、或主进程和负责刷新本地缓存的助手。它们由不同团队做成不同镜像，发布时合成一个 Pod，对外仍是一个服务。

按运行方式，业务可以粗分成四类，分别交给不同控制器：一直在线的用 Deployment，跑完即停的用 Job，每台机器一份的用 DaemonSet，要稳定编号的用 StatefulSet。Pod 是这四类下面的共同零件。

## Pod 怎么实现

两件事先固定：网络怎么共享，文件怎么共享。

### 共享网络

容器默认各自一个网络命名空间，互相看不见对方的端口。Pod 的做法是先起 pause，由它创建网络命名空间、分配 Pod IP 和主机名，再让 nginx、边车等业务容器加入这个命名空间。之后它们监听不同端口，通过 127.0.0.1 互访，对外则共用一个 Pod IP。

步骤可以收成一句：业务容器不单独建网络，而是附着到 pause 已经建好的命名空间上。

### 共享存储

进程写在容器可写层里的文件，容器一换就丢。要让同一 Pod 的两个容器交换文件，在 Pod 上声明卷，再分别挂到各自的路径。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: page-and-sync
  namespace: shop
spec:
  containers:
    - name: nginx
      image: nginx:1.25.4
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
    - name: filler
      image: busybox:1.36
      command: ["sh","-c","echo hello > /work/index.html && sleep 3600"]
      volumeMounts:
        - name: html
          mountPath: /work
  volumes:
    - name: html
      emptyDir: {}
```

`filler` 写入的 `index.html`，nginx 在自己的挂载点上能读到。换成持久卷时，只改 `volumes` 一段，挂载路径可以不动。

## 镜像拉取策略

容器字段 `imagePullPolicy` 决定 kubelet 什么时候向仓库要镜像。

- `IfNotPresent`：节点上已有同名同标签镜像就直接用，这是标签不是 `latest` 时的默认行为
- `Always`：每次创建容器都再向仓库核对，`latest` 默认走这条
- `Never`：只用节点本地镜像，仓库上没有也不拉，适合离线导入之后

```yaml
image: registry.example.net/team-pay/ledger-api:2.3.1
imagePullPolicy: IfNotPresent
```

标签被覆盖推送、节点上又留着旧摘要时，`IfNotPresent` 不会更新。生产里用不可变标签或摘要，并在发布时显式更换。

## 资源限制

调度看的是请求值，不是节点的总核数。容器声明自己至少要 200m CPU 和 256Mi 内存，调度器就只在剩余可分配量够用的节点上落位。上限用来防止它吃满节点。

```yaml
resources:
  requests:
    cpu: "200m"
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi
```

`requests` 参与调度和配额，`limits` 是运行时上限。只写上限不写请求时，请求可能被取成与上限相同，Pod 会变得更难安放。内存超过上限会被杀掉，CPU 超过上限则被限流。

## 重启策略

`restartPolicy` 写在 Pod 上，作用于其中的容器。

- `Always`：无论退出码是什么都重启，常驻服务用这个，也是默认值
- `OnFailure`：只有非 0 退出才重启，成功完成的批处理不应被拉起
- `Never`：退出后保持终止状态，方便查看失败现场

策略决定的是容器还要不要再起，不是 Pod 对象会不会被控制器补一份。控制器发现 Pod 没了，仍可能按模板新建一个。

## 健康检查

`kubectl get pod` 只能说明主进程还在。Java 进程在堆耗尽后可能仍占着端口，却已经不能正确应答。这时要加探针。

- `livenessProbe`：失败就按重启策略杀掉容器
- `readinessProbe`：失败就把 Pod 从 Service 端点里摘掉，容器先不杀

```yaml
livenessProbe:
  httpGet:
    path: /live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
```

三种检查方式：

- `httpGet`：状态码落在 200 到 399 算成功
- `exec`：命令退出码为 0 算成功
- `tcpSocket`：能连上端口算成功，不检查应用协议

存活和就绪不要绑成同一个过重的接口。依赖下游短暂抖动时，就绪失败即可，不必把进程杀掉。

## 调度过程

### Pod 从提交到跑起来

1. 客户端把 Pod 或工作负载提交给 API 进程，对象写入 etcd
2. 调度器发现还没有 `nodeName` 的 Pod，按资源和约束打分，写回绑定
3. 目标节点的 kubelet 看到属于自己的 Pod，调用运行时拉镜像、起容器，并把状态报回去

调度器不负责起容器，kubelet 也不负责给别人的 Pod 挑机器。

### 哪些字段会改变落点

资源请求会先筛掉剩余量不够的节点。

节点选择器要求目标节点带有全部列出的标签。例如测试和预发共用一个集群，预发节点打过 `tier=pre`：

```bash
kubectl label node worker-b tier=pre
```

```yaml
nodeSelector:
  tier: pre
```

没有这个标签的节点不会被考虑。选择器只支持“必须相等”，表达不了或、不等于和权重。

#### 节点亲和

`nodeAffinity` 同样看节点标签，但能写集合运算，并分成硬条件和软条件。

- `requiredDuringSchedulingIgnoredDuringExecution`：必须满足，否则一直 Pending
- `preferredDuringSchedulingIgnoredDuringExecution`：尽量满足，不满足也能调度

常用运算符有 `In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt`。把 `In` 换成 `NotIn` 就是把这组节点排除掉。名称里的 IgnoredDuringExecution 表示：节点标签事后变了，已经跑着的 Pod 不会因此被赶走。

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: tier
              operator: In
              values: ["pre"]
```

## 污点与容忍

### 概述

节点选择器和亲和性是 Pod 上的条件，用来挑选机器。污点写在节点上，表示“不要把普通 Pod 调度过来”，除非 Pod 明确声明自己能容忍。

### 常见用途

- 把某几台机器留给指定业务
- 把带 GPU 或本地 SSD 的机器从通用池里隔开
- 节点故障时用 `NoExecute` 把已有 Pod 赶走

### 查看

```bash
kubectl describe node ctrl-1 | grep -i taint
```

控制面节点通常自带 `node-role.kubernetes.io/control-plane:NoSchedule`，所以业务 Pod 不会落上去。

三种效果：

- `NoSchedule`：新 Pod 不来，已经在上面的不动
- `PreferNoSchedule`：尽量不来，调度器在没别的选择时仍可能使用
- `NoExecute`：新 Pod 不来，并驱逐不能容忍该污点的已有 Pod

### 添加与去掉

```bash
kubectl taint node worker-a dedicated=batch:NoSchedule
kubectl taint node worker-a dedicated=batch:NoSchedule-
```

末尾的 `-` 表示删除这一条，不是把效果改成另一个值。

### 效果演示

先不加污点，创建 4 个副本，观察它们落在哪些工作节点：

```bash
kubectl -n shop create deployment web --image=nginx:1.25.4
kubectl -n shop scale deployment web --replicas=4
kubectl -n shop get pods -o wide
```

副本应分布在未打污点的工作节点上，控制面因为自带污点不会出现在列表里。

给 `worker-a` 打上 `NoSchedule` 后再扩到 6 个。新 Pod 只会去 `worker-b`。已经跑在 `worker-a` 上的不会被这条污点赶走，因为效果不是 `NoExecute`。

```bash
kubectl taint node worker-a dedicated=batch:NoSchedule
kubectl -n shop scale deployment web --replicas=6
kubectl -n shop get pods -o wide
```

实验结束删掉工作负载和污点：

```bash
kubectl -n shop delete deployment web
kubectl taint node worker-a dedicated:NoSchedule-
```

### 容忍

Pod 要进入带污点的节点，就在规格里写 `tolerations`。键、值和效果要与污点对应。`operator: Exists` 可以不写值，表示只要键在就行。

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: batch
    effect: NoSchedule
```

容忍只表示“允许”，不表示“必须”。若还要固定到这台机器，再加节点选择器或亲和性。
