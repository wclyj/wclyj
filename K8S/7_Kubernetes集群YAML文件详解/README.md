# 资源清单 YAML

## 概述

集群里的对象都可以写成声明式文件，再用 kubectl 一次提交。这种文件通常叫资源清单。手敲命令适合试探，重复部署、代码评审和回滚都更适合把期望状态放进清单。

## YAML 文件介绍

### YAML 概述

YAML 是面向数据的文本格式，用缩进表达层级，读起来比同等内容的 JSON 疏一些。Kubernetes 只是它的一个使用方，语法本身不包含 Pod、副本这些概念。

### 基本语法

- 缩进只用空格，同一层左侧对齐
- 空格数量没有唯一规定，但同一层必须一致
- 不要用 Tab，解析器会直接报错
- `#` 后面直到行尾都是注释
- `---` 单独成行，表示后面开始另一份文档，一个文件里可以放多个对象

### 支持的数据结构

#### 对象

对象是一组键值。冒号后面要有空格。

```yaml
site: harbor
port: 8443
inline: {site: harbor, port: 8443}
```

#### 数组

一组以 `- ` 开头的项构成列表，也可以写成一行。

```yaml
zones:
  - east
  - south
zonesInline: [east, south]
```

## 清单由哪几块组成

一份工作负载清单大致分成两段：控制器自己的元数据和规格，以及它管理的 Pod 模板。Pod 模板里才出现镜像、端口和卷。

### 控制器怎么声明

顶层先说明这份文档交给哪个 API、是什么类型、叫什么名字。`spec` 里写期望：要几个副本、用什么标签找到它们、发布时怎么替换。

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
```

`selector` 必须匹配模板上的标签，否则 API 会拒绝创建。

### 被管理的对象

Pod 模板放在 `spec.template` 下面，结构与单独写一个 Pod 几乎相同。

```yaml
  template:
    metadata:
      labels:
        app: notice
    spec:
      containers:
        - name: notice
          image: nginx:1.25.4
          ports:
            - containerPort: 80
```

镜像标签、容器端口、环境变量和资源请求都写在容器这一层，而不是写在 Deployment 的顶层。

### 字段对照

| 字段 | 含义 |
| --- | --- |
| apiVersion | 使用哪一组 API |
| kind | 对象类型，如 Deployment、Service |
| metadata | 名字、命名空间、标签、注解 |
| spec | 期望状态 |
| replicas | 副本个数 |
| selector | 控制器用哪些标签认领 Pod |
| template | Pod 模板 |
| containers | 模板里的容器列表 |

同一份文档里会出现两处 `metadata` 和两处 `spec`：外层属于控制器，内层属于 Pod。改镜像时要改内层。

## 怎样少手写

字段多，从空白文件硬写容易漏选择器。更稳的办法是先生成骨架再改。

### 用 create 生成骨架

`--dry-run=client` 只在本地渲染，不会向集群提交。

```bash
kubectl create deployment notice --image=nginx:1.25.4 --dry-run=client -o yaml > notice.yaml
```

打开 `notice.yaml`，把副本数、资源请求和探针补上，再 `kubectl apply -f notice.yaml`。

### 从已经运行的对象导出

集群里已经有一份来源不明的 Deployment 时，可以把它的规格导出来纳入版本库。

```bash
kubectl -n shop get deploy
kubectl -n shop get deploy notice -o yaml > notice-export.yaml
```

导出结果里常带 `status`、`resourceVersion`、`uid` 和受管字段。这些是当前观察值，提交前应删掉，只保留 `apiVersion`、`kind`、`metadata` 里的名字与标签，以及 `spec`。否则下次 apply 会和集群里的记账字段打架。
