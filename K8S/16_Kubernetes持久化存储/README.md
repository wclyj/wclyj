# 持久化存储

## 前言

`emptyDir` 随 Pod 删除而消失。要在重建后保留文件，目录得放到节点之外。下面先把 NFS 地址写进 Pod 验证挂载，再用 PV 和 PVC 把地址从应用清单里拿出去。

## 步骤

NFS 服务器用 `10.20.8.50`，工作节点要能访问它的 2049 端口。

### 在存储服务器上

安装服务并划出导出目录。

```bash
yum install -y nfs-utils
mkdir -p /srv/exports/shop
```

编辑 `/etc/exports`，只放行集群网段，并允许容器里的 root 写文件：

```text
/srv/exports/shop 10.20.8.0/24(rw,sync,no_root_squash)
```

改完导出表后启动服务：

```bash
systemctl enable --now nfs-server
exportfs -rav
```

### 在工作节点上

会跑 Pod 的节点安装 NFS 客户端，kubelet 才能在本机挂载。

```bash
yum install -y nfs-utils
```

回到 `10.20.8.50` 执行 `systemctl status nfs-server`，确认 2049 已在监听后再写 Pod。

### 在集群里挂到应用

下面这份 Deployment 把页面目录指到 NFS，副本先设为 1。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: page-nfs
  namespace: shop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: page-nfs
  template:
    metadata:
      labels:
        app: page-nfs
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.4
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          nfs:
            server: 10.20.8.50
            path: /srv/exports/shop
```

```bash
kubectl apply -f page-nfs.yaml
kubectl -n shop get pods -l app=page-nfs
```

挂载失败时先核对到 `10.20.8.50` 的网络和 exports。Pod 运行后写入文件再进容器：

```bash
echo 'from-nfs' > /srv/exports/shop/index.html
kubectl -n shop exec -it deploy/page-nfs -- cat /usr/share/nginx/html/index.html
```

读到 `from-nfs` 说明页面目录已经落在 `10.20.8.50:/srv/exports/shop`。

## PV 和 PVC

地址若写死在每个 Deployment 里，换盘就要改所有应用。PV 由管理员登记，PVC 由业务申请容量，Pod 只写 PVC 的名字。

### 绑定顺序

1. 管理员创建 PV，写明容量、访问模式、回收策略和真实后端
2. 业务创建 PVC，写明需要的容量和访问模式
3. 控制面把两者绑在一起；Pod 再引用 PVC

容量或访问模式对不上时，PVC 会停在 Pending。

### 示例

PVC 与 Deployment 放在同一文件，副本改为 2。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: page-data
  namespace: shop
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: page-pvc
  namespace: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: page-pvc
  template:
    metadata:
      labels:
        app: page-pvc
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.4
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          persistentVolumeClaim:
            claimName: page-data
```

PV 不属于命名空间，按容量和访问模式与 PVC 匹配。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-shop-2g
spec:
  capacity:
    storage: 2Gi
  accessModes: ["ReadWriteMany"]
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.20.8.50
    path: /srv/exports/shop
```

先提交 PV 再提交 PVC。

```bash
kubectl apply -f nfs-shop-pv.yaml
kubectl apply -f page-pvc.yaml
kubectl get pv,pvc -n shop
kubectl -n shop exec -it deploy/page-pvc -- cat /usr/share/nginx/html/index.html
```

两者变为 Bound，容器里仍能读到 `from-nfs`。路径没变，只是 IP 和导出目录留在 PV 里。
