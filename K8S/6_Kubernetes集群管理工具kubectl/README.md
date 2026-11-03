# kubectl：从命令行操作集群

## 它管什么

kubectl 把本地请求发给 apiserver。集群本身的节点、证书、污点，以及业务的工作负载、服务、配置，都通过同一套子命令查看或变更。它不负责在节点上拉镜像，那是 kubelet 和容器运行时的事。

## 写法

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

- `command`：动作，例如 `create`、`get`、`describe`、`delete`。
- `TYPE`：资源种类。大小写敏感，但同一种资源通常接受单数、复数和短名。
- `NAME`：对象名字，大小写敏感。省掉名字时列出该种类的全部对象。
- `flags`：可选项，例如 `-n` 指定命名空间，`-o yaml` 改输出，`--server` 临时指定 apiserver 地址。

同一次查询的三种写法等价，短名只是少打几个字母：

```bash
kubectl get pod bulletin
kubectl get pods bulletin
kubectl get po bulletin
```

省略名字则列出当前命名空间里所有 Pod：

```bash
kubectl get pods
```

默认命名空间是 `default`。系统组件在 `kube-system`，要加 `-n`，或者用 `-A` 跨命名空间。

## 先会求助

```bash
kubectl --help
kubectl explain deployment.spec.replicas
kubectl get --help
```

`explain` 读的是 apiserver 的 OpenAPI，字段解释比 `--help` 更细，写清单时优先用它确认字段是在 `spec` 还是 `spec.template.spec`。

## 日常会用到的子命令

### 创建与查看

| 子命令 | 做什么 |
| --- | --- |
| create | 按清单或参数新建对象，已存在则报错 |
| expose | 给工作负载配一个 Service |
| run | 用指定镜像起一个临时 Pod |
| set | 改已有对象的镜像、环境变量等单项 |
| get | 列出对象的摘要 |
| explain | 查看资源字段说明 |
| edit | 用本地编辑器改服务端对象 |
| delete | 按名字、清单或标签删除 |

`create` 遇到同名对象会失败；想按文件把集群改成文件里的样子，用后面的 `apply`。

### 副本与发布

| 子命令 | 做什么 |
| --- | --- |
| rollout | 看发布状态、暂停、恢复或退回 |
| scale | 改 Deployment、ReplicaSet 或 StatefulSet 的副本数 |
| autoscale | 建 HPA，按指标调整副本 |

`rolling-update` 只作用于早已淘汰的 ReplicationController，新的工作负载用 `rollout`。

### 节点维护

| 子命令 | 做什么 |
| --- | --- |
| certificate | 处理证书签名请求 |
| cluster-info | 打印控制面地址 |
| top | 看节点或 Pod 的 CPU、内存，依赖 metrics-server |
| cordon | 标记节点不再接收新 Pod |
| uncordon | 取消上述标记 |
| drain | 赶走节点上的 Pod，为关机维护做准备 |
| taint | 给节点加污点或去掉污点 |

维护顺序一般是 `cordon`、`drain`、关机，恢复后再 `uncordon`。跳过 `drain` 直接关机，上面的 Pod 要等超时才会在别的节点重建。

### 排障

| 子命令 | 做什么 |
| --- | --- |
| describe | 看事件、状态条件和最近的探针失败 |
| logs | 打容器标准输出；多容器 Pod 要加 `-c` |
| exec | 在容器里执行一条命令 |
| attach | 连到容器已有的标准输入输出 |
| port-forward | 把本地端口转到 Pod 或 Service |
| proxy | 在本机开一个到 apiserver 的代理 |
| cp | 在本地和容器之间拷文件 |
| auth | 检查当前身份能否对某个资源做某个动作 |

Pod 起不来时先 `describe` 看 Events，再 `logs --previous` 看上一次崩溃前的输出。

### 声明式修改及其他

| 子命令 | 做什么 |
| --- | --- |
| apply | 按文件创建或更新，并记录上次应用的字段 |
| patch | 只改若干字段，战略合并、JSON 补丁或 JSON merge 均可 |
| replace | 用文件整体替换对象 |
| label / annotate | 改标签或注解 |
| config | 管理 kubeconfig 里的集群、用户和上下文 |
| api-resources / api-versions | 列出集群认识的资源和 API 组 |
| version | 同时看客户端和 apiserver 的版本 |
| completion | 给 shell 生成补全脚本 |

## 一组最小闭环

下面起一个公告页，镜像是 `httpd:2.4-alpine`，Service 类型为 NodePort，名字用 `bulletin`：

```bash
kubectl create deployment bulletin --image=httpd:2.4-alpine
kubectl expose deployment bulletin --port=8080 --target-port=80 --type=NodePort
kubectl get deploy,po,svc -l app=bulletin
kubectl rollout status deployment/bulletin
```

`get` 里 Service 的端口列会出现一个 30000 以上的节点端口。若要删掉这组对象：

```bash
kubectl delete svc bulletin
kubectl delete deployment bulletin
```

只删 Deployment 不会删掉 `expose` 创建的 Service，选择器会变成空集合，地址仍占用着。

## 常见问题

- 忘了 `-n`，在 `default` 里找 `kube-system` 的对象，会误以为组件丢失。
- `kubectl edit` 改完又用旧文件 `apply`，手工修改被覆盖。长期变更应写回文件。
- 客户端比 apiserver 新一个小版本通常没问题，跨两个小版本以上时，先升集群再升 kubectl。
