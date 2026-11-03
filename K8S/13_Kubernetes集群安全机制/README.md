# 集群安全机制

## 概述

请求进入 API 进程后过三关：认证确认身份，鉴权检查动作，准入决定对象能不能写入。

人使用证书、令牌或用户名口令；Pod 使用 ServiceAccount。对外只开放 6443 上的 HTTPS。

### 认证

常见是集群 CA 签发的客户端证书、HTTP 令牌，以及用户名加口令。认证通过还不代表拥有资源权限。

### 鉴权

按 RBAC 判断主体能否对某命名空间里的资源执行该动词。没有绑定就是拒绝。

### 准入控制

准入插件可以改写或拒绝已通过鉴权的对象，例如限制镜像来源、补默认值、禁止特权容器。启用哪些插件由 API 进程的启动参数决定。

## RBAC 介绍

先把允许的动作写成角色，再绑到主体。

- Role 只在一个命名空间生效；ClusterRole 面向全集群
- RoleBinding 在某个命名空间把角色交给主体；引用 ClusterRole 时，效果仍停在该空间
- ClusterRoleBinding 没有命名空间限制
- 主体是 User、Group 或 ServiceAccount

## RBAC 实现鉴权

在独立命名空间里只授予 Pod 的 get 和 list。

### 创建命名空间

```bash
kubectl get ns
kubectl create ns auth-lab
```

### 在该空间运行 Pod

不写 `-n` 会进 default，和角色不在同一空间。

```bash
kubectl -n auth-lab run webbox --image=nginx:1.25.4
```

### 创建角色

`pod-reader` 只有 Pod 的 get 和 list。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: auth-lab
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

### 创建角色绑定

用户名 `lin` 必须和证书 CN 一致。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: lin-reads-pods
  namespace: auth-lab
subjects:
  - kind: User
    name: lin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f pod-reader-role.yaml -f lin-binding.yaml
kubectl -n auth-lab get role,rolebinding
```

### 用证书识别身份

用集群 CA 给 CN 为 `lin` 的证书签名，写进单独的 `lin.kubeconfig`。

```bash
openssl genrsa -out lin.key 2048
openssl req -new -key lin.key -subj "/CN=lin" -out lin.csr
openssl x509 -req -in lin.csr -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out lin.crt -days 180
kubectl --kubeconfig=lin.kubeconfig config set-credentials lin \
  --client-certificate=lin.crt --client-key=lin.key --embed-certs=true
kubectl --kubeconfig=lin.kubeconfig config set-cluster lab \
  --server=https://10.20.8.11:6443 \
  --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true
kubectl --kubeconfig=lin.kubeconfig config set-context lin@lab --cluster=lab --user=lin
kubectl --kubeconfig=lin.kubeconfig config use-context lin@lab
kubectl --kubeconfig=lin.kubeconfig -n auth-lab get pods
kubectl --kubeconfig=lin.kubeconfig -n auth-lab get svc
```

第一条能看到 `webbox`，第二条会被拒绝。
