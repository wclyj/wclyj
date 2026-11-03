# 容器运行时接口 CRI

## 前言

节点上真正创建、停止容器的是容器运行时，不是 apiserver。Docker Engine 曾经是最常见的选择，但不是唯一选择。为了让 kubelet 不必为每一种运行时各维护一套调用代码，Kubernetes 把“拉镜像、建沙箱、起容器、收日志”收成一组 gRPC 接口，这就是容器运行时接口（Container Runtime Interface，CRI）。运行时只要实现这组接口，kubelet 就可以通过本地套接字使用它。

## 一个 Pod 从提交到落地

在节点上创建业务之前，集群至少要有控制面和工作节点。下面用一份清单把页面 Pod 交进去：

```bash
kubectl apply -f notice-page.yaml
```

请求先到 kube-apiserver。apiserver 做完认证、鉴权和准入后，把对象写入 etcd。控制器管理器的 ReplicaSet 控制循环发现副本不够，会补上 Pod 对象，此时 Pod 还没有 `nodeName`。

调度器监听的是 apiserver，不是直接打开 etcd。它看到未绑定的 Pod 后，按资源请求、污点、亲和性和已有负载选出一台节点，把节点名写回 Pod。这一步称为绑定。

工作节点上驻留的是 kubelet，不是 kubectl。kubectl 只是管理员手里的客户端，跑完命令就退出。kubelet 通过 apiserver 的 watch 看到“绑定到本机”的 Pod，才开始拉镜像、调 CRI 建沙箱和容器。kubelet 也不直接读 etcd，所有读写都走 apiserver，这样鉴权和审计不会被绕开。

## 早期调用链：经过 dockershim

Kubernetes 1.24 之前，默认安装里 kubelet 内置 dockershim，调用链大致是：

1. kubelet 作为 CRI 客户端，调用 dockershim 这个 CRI 服务端，请求为一个 Pod 准备环境。
2. dockershim 把 CRI 请求翻译成 Docker Engine 的 API，交给 dockerd。
3. Docker 1.12 之后，dockerd 把容器生命周期委托给 containerd。
4. containerd 不为每个容器长期充当父进程，而是拉起一个 containerd-shim。
5. shim 调用 runC。runC 按 OCI 运行时规范创建 Linux namespace 和 cgroup，挂上根文件系统，然后自己退出。
6. shim 留下来，成为容器进程的父进程：回收退出状态、保持标准流的文件描述符、在一号进程退出后清掉孤儿进程。dockerd 或 containerd 短时重启时，容器不会因为父进程消失而被系统杀掉。

shim 存在的几个具体原因：

- 收集容器状态，并替容器拿着 stdin 等需要保持打开的描述符。
- runC 创建完就可以退出，不必每个容器驻留一个运行时进程。
- 引擎进程重启期间，容器的标准流仍然可用。
- 把退出码回报给 containerd。
- 升级 dockerd 时不必中断已经在跑的容器。

1.24 起 kubelet 删除了内置 dockershim。继续使用 Docker Engine 需要单独的 cri-dockerd；更常见的做法是 kubelet 直接对话 containerd 或 CRI-O 自带的 CRI 插件，链路上不再出现 dockerd。

## 这条链为什么变长

最早 kubelet 直接调用 Docker API，dockerd 再用 libcontainer 建容器，中间没有 CRI，也没有单独的 shim。后来发生了三件叠在一起的事。

第一，为避免单一实现绑死接口，社区制定了开放容器标准 OCI。Docker 把 libcontainer 整理成 runC，作为 OCI 运行时的参考实现捐出。镜像布局则由 OCI Image Spec 描述。CRI 解决“kubelet 如何叫运行时”，OCI 解决“运行时如何把进程隔离出来”，两者不是同一层。

第二，Docker 把容器操作下沉到 containerd，让 dockerd 专做镜像构建和上层体验，并曾用 Swarm 争夺编排市场。Swarm 没有成为主流之后，containerd 进入 CNCF，成为可独立部署的运行时。

第三，当时还有 rkt 希望被 kubelet 一等支持，虚拟机隔离的 runV（后来并入 Kata Containers）也需要接入。每接一种运行时就在 kubelet 里加一段专用代码，SIG-Node 的维护成本线性上涨。于是把容器操作收成稳定的 gRPC：kubelet 只认 CRI，运行时各自提供一个 shim 监听本地套接字。rkt 专用代码最终被移除，接口留了下来。

## CRI 上的两类服务

CRI 基于 gRPC，分成两个服务。运行时在节点上监听 Unix 套接字（Windows 上用命名管道或 TCP），kubelet 启动参数 `--container-runtime-endpoint` 指向它。

### RuntimeService：Pod 沙箱与容器

Pod 在 CRI 里先有一个沙箱（PodSandbox），网络和 Linux 命名空间通常挂在沙箱上，业务容器再放进沙箱。

| RPC | 作用 |
| --- | --- |
| Version | 返回运行时名称、版本和 CRI 版本 |
| RunPodSandbox | 创建并启动沙箱，成功时沙箱必须已处于就绪 |
| StopPodSandbox | 停掉沙箱内进程并收回 IP 等网络资源；可重复调用 |
| RemovePodSandbox | 删除沙箱；若仍有容器，先强制去掉 |
| PodSandboxStatus / ListPodSandbox | 查询一个或一组沙箱 |
| CreateContainer / StartContainer | 在指定沙箱里创建容器，再单独启动 |
| StopContainer | 按宽限时间停止；已经停止时不要返回错误 |
| RemoveContainer | 删除容器；仍在运行则强制删除 |
| ListContainers / ContainerStatus | 按条件列出容器或读取单个状态 |
| UpdateContainerResources | 调整容器的 cgroup 资源 |
| ReopenContainerLog | 日志轮转后重新打开 stdout/stderr 文件 |
| ExecSync | 在容器内同步执行命令并等结果 |
| Exec / Attach / PortForward | 准备流式终端、附着或端口转发的地址 |
| ContainerStats / ListContainerStats | 读取单个或全部容器的资源统计 |
| UpdateRuntimeConfig | 更新运行时配置 |
| Status | 报告运行时是否健康 |

几个调用要幂等：kubelet 可能因为超时而重试 `StopPodSandbox` 或 `RemoveContainer`。运行时如果在资源已经清理后再返回错误，kubelet 会把 Pod 卡在反复删除里。

`CreateContainer` 与 `StartContainer` 分开，是为了让 kubelet 在启动前做完卷挂载和钩子。只创建不启动时，容器处于已创建、未运行。

### ImageService：镜像

| RPC | 作用 |
| --- | --- |
| ListImages | 列出本机镜像 |
| ImageStatus | 查询单个镜像；不存在时状态为空，而不是一律失败 |
| PullImage | 按仓库认证信息拉取 |
| RemoveImage | 删除镜像；已经不在时视为成功 |
| ImageFsInfo | 返回存放镜像的文件系统用量 |

镜像服务和容器服务拆开，是因为同一节点上的多个 Pod 共享镜像存储，生命周期并不跟随某一个容器。

业务运行时（containerd、CRI-O 等）实现上述服务，进程通常就叫 CRI shim。kubelet 不链接这些运行时的私有库，只通过套接字发 gRPC。因此升级运行时，只要套接字和 CRI 版本仍被 kubelet 接受，就不必重编译 kubelet。

## 和现在节点的对应关系

以 containerd 为例，当前常见路径是：

1. kubelet 调用 containerd 的 CRI 插件（`/run/containerd/containerd.sock`）。
2. 插件让 containerd 准备沙箱和容器任务。
3. containerd-shim 再调用 runc 或其他 OCI 运行时（例如支持虚拟机隔离的 Kata）。
4. runc 退出后，shim 继续监管容器进程。

可以用下面的命令确认 kubelet 连的是哪一个套接字，而不是凭进程列表里有没有 dockerd 来猜测：

```bash
ps -ef | awk '/kubelet/ && !/awk/'
crictl --runtime-endpoint unix:///run/containerd/containerd.sock version
crictl ps
```

`crictl` 是 CRI 的命令行客户端，看到的对象是沙箱和容器，不是 Docker 的容器视图。只装了 Docker CLI 时，`docker ps` 可能看不到 containerd 直接拉起的工作负载。

## 从一份清单看到接口怎么被调用

下面这份页面只跑一个容器，用来对照 CRI 的调用顺序。名字用 `notice-page`，镜像固定标签，并声明就绪探针，避免容器刚创建就被当成可接流量。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: notice-page
  namespace: lab
spec:
  containers:
    - name: httpd
      image: httpd:2.4.58-alpine
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 2
```

`kubectl apply` 之后，kubelet 不会直接 `CreateContainer`。顺序是：

1. `RunPodSandbox` 先建立 Pod 级沙箱。沙箱里通常有一个 pause 容器，用来拿住网络命名空间和 IP。CNI 插件在这一步被运行时调用，把 `10.90.0.0/16` 这类地址配上 veth。沙箱失败时，业务容器不会出现。
2. `PullImage` 按 `imagePullSecrets` 里的仓库认证拉取 `httpd:2.4.58-alpine`。本地已有相同摘要则跳过真正下载，但状态仍要能被 `ImageStatus` 查到。
3. `CreateContainer` 写入挂载、环境变量和探针定义，此时进程还没开始。
4. `StartContainer` 才让 OCI 运行时把进程放进沙箱的命名空间。
5. kubelet 周期调用 `ContainerStatus`。就绪探针没过时，Pod 条件不是 Ready，Service 不会把地址放进后端。
6. 删除 Pod 时先 `StopContainer`（尊重 `terminationGracePeriodSeconds`），再 `StopPodSandbox` 回收 IP，最后 `RemoveContainer` 与 `RemovePodSandbox`。每一步都可能被重试。

因此排障要分清卡在哪一层：镜像拉不下来是 ImageService；一直 ContainerCreating 且事件写 sandbox 是 RuntimeService 的沙箱或 CNI；容器反复退出才是进程本身。

## CRI、OCI、CNI 各管一段

三套接口容易被说成一件事，其实交接的对象不同。

- CRI 是 kubelet 对运行时说的话：要一个沙箱、一个容器、一次拉取。传输是本机 gRPC。
- OCI 运行时规范是运行时对 runc 这类工具说的话：给定一个 `config.json`，把进程隔离出来。runc 不认识 Pod，也不知道 Service。
- CNI 是给沙箱配网的插件协议。运行时在 `RunPodSandbox` 期间执行 CNI 二进制，把结果里的地址回填到沙箱状态。kubelet 不直接调用 CNI 二进制。

所以换 CNI 插件不用改业务清单，换 runc 到 Kata 也不用改 kubelet 代码，只要 CRI 实现仍然报告同一套状态。换运行时则要改 kubelet 的套接字参数，并确认两边支持的 CRI 版本有交集。`crictl version` 里的 RuntimeApiVersion 就是这个交集会面。

## 在节点上核对

控制面看事件，节点上看运行时是否真的建出了沙箱。下面假设套接字是 containerd 的默认路径：

```bash
crictl --runtime-endpoint unix:///run/containerd/containerd.sock pods
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps --name httpd
crictl --runtime-endpoint unix:///run/containerd/containerd.sock inspectp <沙箱ID>
```

`pods` 对应沙箱，`ps` 对应容器。只有容器没有沙箱，说明清理顺序乱了；只有沙箱没有容器，说明卡在拉取或 `CreateContainer`。`inspectp` 里能看到 CNI 返回的地址，若为空，就不要先去改 Deployment 的标签。

kubelet 日志里如果是 `connection refused`，多半是套接字路径或权限，而不是镜像名写错。若使用 cri-dockerd，套接字常常在 `/var/run/cri-dockerd.sock`，和 containerd 的路径不同，写错一个就会让节点上所有新 Pod 停住。

## 常见问题

- 把 kubectl 当成节点代理。kubectl 不 watch 对象，也不会在节点上拉镜像。
- 1.24 之后仍按旧文档给 kubelet 加 `--container-runtime=docker`，参数已被删除，kubelet 会拒绝启动。
- CRI 套接字权限不对，kubelet 日志里是连接被拒绝，Pod 事件却只写 “sandbox failed”，要两边一起看。
- 只实现了容器 RPC、没有正确实现 `StopPodSandbox` 的幂等，删除命名空间时会留下占着 IP 的沙箱。
