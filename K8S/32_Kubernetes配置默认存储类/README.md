# 给集群配一个默认 StorageClass

## 现象从哪来

安装带持久化的平台组件时，安装器会创建 PVC，并指望集群里存在默认存储类。若 `kubectl get storageclass` 什么都不返回，PVC 会停在 Pending，事件里通常是“没有可用的 provisioner”或“未设置默认 StorageClass”。有状态的 Pod 跟着起不来，看起来像应用故障，实际是存储没接上。

下面用一台独立的 NFS 服务器做动态供给，并把它标成默认存储类。实验地址是 `10.6.4.21`，导出目录是 `/export/pv-share`。

## 准备 NFS 服务端

在 `10.6.4.21` 上安装工具并建立导出目录。这台机器不要和业务节点混用磁盘，避免节点排空时把存储也停掉。

```bash
yum install -y nfs-utils
mkdir -p /export/pv-share
```

编辑 `/etc/exports`，只允许集群节点所在网段挂载，不要对所有地址开放：

```text
/export/pv-share 10.6.4.0/24(rw,sync,no_subtree_check,no_root_squash)
```

`no_root_squash` 让容器里的 root 写入不会被映射成 nfsnobody，很多镜像以 root 写数据目录时需要它。它也放宽了权限，导出范围必须收到集群网段。

```bash
exportfs -rav
systemctl enable --now nfs-server
```

## 工作节点安装客户端

每一台会调度业务 Pod 的节点都要能挂载 NFS，否则 Pod 被分到没装客户端的节点时，卷挂载阶段失败：

```bash
yum install -y nfs-utils
showmount -e 10.6.4.21
```

`showmount` 能列出 `/export/pv-share` 再继续。只在节点上安装客户端，不要在节点上再次 `exportfs` 同一目录，避免弄成环。

## 动态供给器做什么

StorageClass 自己不会创建目录。还要一个 provisioner：有 PVC 时它在 NFS 上建子目录并生成 PV，删除时再按回收策略处理。子目录常用「命名空间-PVC 名-PV 名」这种拼法；若回收策略是 Retain 或供给器配置了归档，旧目录会改名保留而不是立刻删掉。

供给器使用的镜像与环境变量要和 StorageClass 的 `provisioner` 字段一致。下面用命名空间 `storage-infra`，供给器标识 `lab.local/nfs`。

### Deployment

`nfs-provisioner.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-dyn-provisioner
  namespace: storage-infra
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-dyn-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-dyn-provisioner
    spec:
      serviceAccountName: nfs-dyn-provisioner
      containers:
        - name: nfs-dyn-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:v4.0.2
          volumeMounts:
            - name: export-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: lab.local/nfs
            - name: NFS_SERVER
              value: "10.6.4.21"
            - name: NFS_PATH
              value: /export/pv-share
      volumes:
        - name: export-root
          nfs:
            server: 10.6.4.21
            path: /export/pv-share
```

`strategy: Recreate` 保证旧副本先退出再起新的，避免两个供给器同时处理同一个 PVC。镜像标签写死，不用 `latest`。

### 账号与权限

供给器要能创建和删除 PV，也能观察 PVC。权限收在 ClusterRole 里，再绑到 `storage-infra` 这个命名空间的 ServiceAccount。`nfs-rbac.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: storage-infra
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-dyn-provisioner
  namespace: storage-infra
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-dyn-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nfs-dyn-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-dyn-provisioner
    namespace: storage-infra
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nfs-dyn-provisioner
```

少了 PV 的 `create`，供给器日志里会是 Forbidden，PVC 仍 Pending，但 StorageClass 对象看起来是正常的。

### StorageClass

`nfs-sc.yaml` 的 `provisioner` 必须等于 Deployment 里的 `PROVISIONER_NAME`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: lab-nfs-sc
provisioner: lab.local/nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

## 创建并设为默认

先有命名空间和 RBAC，再起 Deployment，最后才是 StorageClass。顺序反了，供给器会因为找不到 ServiceAccount 起不来。

```bash
kubectl apply -f nfs-rbac.yaml
kubectl apply -f nfs-provisioner.yaml
kubectl apply -f nfs-sc.yaml
kubectl -n storage-infra get pods
kubectl get storageclass
```

Pod 为 Running 后，把 `lab-nfs-sc` 标成默认。集群里只应有一个默认类：

```bash
kubectl patch storageclass lab-nfs-sc -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass
```

名字旁边出现 `(default)` 即表示不写 `storageClassName` 的 PVC 会走它。

用一份不指定存储类的 PVC 做验收：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-cache
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f demo-cache.yaml
kubectl get pvc demo-cache
```

STATUS 变成 Bound，并且在 `10.6.4.21:/export/pv-share` 下出现新目录，说明供给链路通了。

## 常见问题

- NFS 服务器防火墙拦了 2049，节点 `showmount` 失败，供给器 Pod 停在 ContainerCreating。
- `provisioner` 字符串和环境变量差一个字符，StorageClass 存在，但永远不会生成 PV。
- 两份 StorageClass 都标了 default，调度器用哪一个不确定，验收时以 `kubectl get storageclass` 里唯一的 `(default)` 为准。
- `no_root_squash` 配合过宽的导出网段，等于把数据目录暴露给整网的 root，实验结束后把网段收回来。
