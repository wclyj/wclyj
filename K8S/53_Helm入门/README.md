# Helm 入门：把发布当成一次版本化操作

官方手册：<https://helm.sh/zh/docs/>

## 清单多了以后会怎样

Kubernetes 上的应用由多种对象拼成，Deployment、Service、Ingress、配置各自一份文件，或者挤在同一个文件里用 `---` 分隔。对象少的时候，`kubectl apply -f` 足够。

服务一旦拆成十几个进程，每次发布都要指出改了哪些文件、能否整包退回。只靠目录里的 YAML 很难回答三个问题：这些对象算不算同一个应用，换环境时哪些字段可以复用，上一次成功的发布到底是哪一版。Helm 用 Chart 回答前两个问题，用 Release 的修订号回答第三个。

## 名称

Helm 类似系统软件包工具：作者把清单打成包，使用者用名字安装，而不是逐个文件 kubectl。

- Helm：本机命令行，负责 Chart 的创建、打包、安装和升级。
- Chart：尚未安装的应用包，包含模板、默认值和依赖说明。
- Release：Chart 安装到某个命名空间后的实例。同一 Chart 安装两次，就是两个 Release，各自有历史。

Helm 3 不再包含集群内的 Tiller。2019-11-13 的第一个 Helm 3 稳定版把渲染和提交都放回客户端，Release 记录存在它所在的命名空间，因此不同命名空间可以复用同一个 Release 名。后续版本还补上了把 Chart 推到 OCI 仓库，以及用 JSON Schema 校验 values。

## 安装客户端

发行文件在 <https://github.com/helm/helm/releases> 。选择与集群小版本相匹配的 Helm 3，例如 v3.14.4，解压后把 `helm` 放到 `/usr/local/bin`。`helm version` 应只打印客户端信息，不会再出现 Tiller 地址。

## 常用子命令

| 子命令 | 用途 |
| --- | --- |
| create | 生成一个新的 Chart 目录 |
| dependency | 拉取或更新 Chart 依赖 |
| install | 安装 Chart，生成 Release |
| upgrade | 用新 values 或新 Chart 更新 Release |
| rollback | 按修订号退回 |
| uninstall | 删除 Release 及其管理的对象 |
| list / history / status | 看 Release、修订和本次说明 |
| get | 取出某次发布的清单、values、钩子或备注 |
| package | 把 Chart 目录打成 `.tgz` |
| pull | 从仓库下载 Chart，`--untar` 可展开 |
| repo | 添加、列出、删除、更新仓库索引 |
| search | 在已添加仓库中按关键字查找 |
| show | 看 Chart 的说明、默认 values 或 Chart.yaml |
| template | 只在本地渲染，不接触集群 |
| version | 打印客户端版本 |

## 配置仓库

公共 Chart 不随客户端内置，要先添加仓库再更新索引：

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add community https://mirror.azure.cn/kubernetes/charts
helm repo update
helm repo list
helm search repo bitnami/redis
```

索引过期时 `search` 结果为空，先 `repo update`。确认不再使用的源：

```bash
helm repo remove community
```

旧文档里的 `stable` 仓库已经停止更新，包名还在，镜像和 Kubernetes API 版本却可能过时。新环境优先用仍在维护的仓库。

## 安装、升级、回退

查找 Redis 这个 Chart，并在安装前看默认参数：

```bash
helm search repo bitnami/redis
helm show values bitnami/redis | head
```

搜索关键字是 `redis` 时，名称或描述里带这个词的 Chart 都会出现，不代表它们互相替代。安装到命名空间 `data`，Release 名用 `session-cache`：

```bash
helm install session-cache bitnami/redis \
  --namespace data --create-namespace
kubectl -n data get pods
kubectl -n data describe pod -l app.kubernetes.io/instance=session-cache
```

很多 Chart 默认要求持久卷。集群没有默认 StorageClass 时，Pod 会停在 Pending，事件写的是 PVC 未绑定。这不是 Helm 安装失败，而是 values 里的存储类在当前集群不存在。

覆盖参数有两种方式，可以同时用。`-f` 指定文件，可以写多次，右边的文件覆盖左边；`--set` 写在命令行上，优先级高于文件。

`cache-values.yaml`：

```yaml
architecture: standalone
auth:
  enabled: true
  password: "cache-pass-6380"
master:
  persistence:
    storageClass: lab-nfs-sc
    size: 2Gi
replica:
  replicaCount: 0
```

```bash
helm install session-cache bitnami/redis \
  -n data -f cache-values.yaml \
  --set master.persistence.size=4Gi
```

这次会创建启用认证的单节点 Redis，数据卷向 `lab-nfs-sc` 申请 4Gi，因为 `--set` 盖过了文件里的 2Gi。口令不要提交进公共仓库，实验结束后改掉。

只想把包拿下来阅读：

```bash
helm pull bitnami/redis --untar --version 18.19.0
```

指定版本可以避免今天和明天拉到不同的 Chart。

升级与回退：

```bash
helm upgrade session-cache bitnami/redis -n data -f cache-values.yaml
helm history session-cache -n data
helm rollback session-cache 1 -n data
```

`history` 里修订号 1 是第一次安装，2 是升级。回退到 1 会再生成一条新修订，而不是把历史抹掉。

## 自己做一个 Chart

```bash
helm create quotechart
```

生成的目录里，`Chart.yaml` 是包名和版本，`values.yaml` 是默认参数，`templates/` 放 Deployment 和 Service 模板，`templates/NOTES.txt` 是安装结束后打印给操作者的说明。把仓库地址、镜像标签这些会变的内容留在 values，模板只保留结构，换环境时才不用复制整棵目录。

本地试渲染：

```bash
helm template quote ./quotechart
helm lint ./quotechart
```

`lint` 能抓住未闭合的模板和明显的 YAML 错误。渲染结果确认后，再 `helm install quote ./quotechart -n data`。

## 常见问题

- 命令写成 `helm repo repo` 或把客户端叫成别的拼写，都不是有效子命令。仓库操作是 `helm repo list`。
- `--set key=value` 里值含逗号或花括号时会被再解析，复杂字符串改用 `-f`。
- 卸载 Release 默认删除它创建的对象，PVC 是否跟随取决于 Chart 是否把卷声明放进钩子或注解，卸载前先 `helm get manifest` 看一眼。
