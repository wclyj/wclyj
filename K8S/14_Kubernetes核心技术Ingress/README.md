# Ingress：用域名把流量分到不同 Service

## 前言

NodePort 靠节点 IP 加高位端口对外，端口不能重复。Ingress 改按主机名转发，入口只留 80 与 443。

## 与 Service、Pod 的关系

Ingress 不直接选 Pod。`shop-svc` 用标签找到 Pod，规则再把 `shop.lab.local` 指到 8080。控制器负责转发。改副本只动 Deployment，改域名只动 Ingress。

## 请求怎么走

`shop.lab.local` 进 `shop-svc`，`pay.lab.local` 进 `pay-svc`，两个名字共用入口地址。Ingress 只是规则，监听端口的控制器要单独安装。

## 操作步骤

### 准备后端

在 `edge-apps` 中用 `httpd:2.4-alpine` 建两个站点，容器端口 80，Service 端口 8080。`--type` 必须用等号。

```bash
kubectl create ns edge-apps
kubectl -n edge-apps create deploy shop --image=httpd:2.4-alpine
kubectl -n edge-apps expose deploy shop --name=shop-svc --port=8080 --target-port=80
kubectl -n edge-apps create deploy pay --image=httpd:2.4-alpine
kubectl -n edge-apps expose deploy pay --name=pay-svc --port=8080 --target-port=80
```

### 安装控制器

控制器装到 `ingress-nginx`。要用节点 IP 访问时打开 `hostNetwork: true`，并设置 `dnsPolicy: ClusterFirstWithHostNet`。

```bash
kubectl apply -f ingress-nginx-deploy.yaml
kubectl get pods -n ingress-nginx
```

Pod 为 Running 后再写规则。

### 编写规则

`ingressClassName` 必须与控制器的 class 一致，否则规则无人认领。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-rules
  namespace: edge-apps
spec:
  ingressClassName: nginx
  rules:
  - host: shop.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-svc
            port: {number: 8080}
  - host: pay.lab.local
    http:
      paths:
      - path: /callback
        pathType: Prefix
        backend:
          service:
            name: pay-svc
            port: {number: 8080}
```

```bash
kubectl apply -f shop-rules.yaml
kubectl -n edge-apps describe ingress shop-rules
```

### 本机域名

没有公网解析时，写入本机 hosts：

```text
10.20.1.21  shop.lab.local
10.20.1.21  pay.lab.local
```

```bash
curl -sI http://shop.lab.local/
curl -sI http://pay.lab.local/callback
```

## 常见问题

- 没装控制器时，对象在但端口不开。
- 规则与 Service 跨命名空间时，后端无效。
