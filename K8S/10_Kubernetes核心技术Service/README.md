# Service：给一组 Pod 一个稳定入口

## 前言

Deployment 只维持副本数量。副本仍会在节点之间重建，新 Pod 拿到新的集群地址。调用方若写死某个 Pod IP，下一次滚动更新就会连空。

稳定访问要同时满足两件事：用固定名字找到后端，并把流量分到仍健康的副本。Service 持有一个只在集群内路由的虚拟地址，kube-proxy 在每个节点上把发往该地址的报文改写到真实 Pod。节点变多时转发点也变多，不必再为均衡器本身做主备。

## Service 解决什么问题

### 副本地址会变

每次创建 Pod，CNI 都会分一个新地址，旧地址随 Pod 删除立即作废。页面有多份副本、接口也有多份副本时，若让页面自己登记接口 IP，服务发现就退回人工维护。Service 用标签筛出当前匹配的后端，调用方只访问 Service 的名字或 ClusterIP。

### 流量如何分到副本

请求先到达 Service，再由本机 kube-proxy 选择后端。iptables 模式按概率挑选；IPVS 模式可以换成轮询、加权轮询或最少连接。教学里说的“随机、轮询、响应比”只是直观比喻，真正的算法取决于 kube-proxy 的模式，不是 Service 清单里的一个字段。

## Pod 与 Service 如何绑定

绑定靠标签，不靠名字。Pod 模板写 `app: catalog`，Service 的 `selector` 写同一组键值，EndpointSlice 里就会出现这些 Pod 的 IP 和容器端口。一个标签写错时，Service 对象仍在，但后端为空，访问超时。

调用方使用的是虚拟 IP，不是任何一个 Pod 的地址。该地址落在初始化时划定的 Service 网段，不能和宿主机网段、Pod 网段重叠。

## 三种常用类型

- ClusterIP：仅集群内访问，也是省略 `type` 时的默认值。
- NodePort：在每个节点上打开同一个高位端口，适合实验环境临时对外开放。
- LoadBalancer：交给云控制器分配外部地址。没有云控制器的机房里，这个类型会一直 Pending。

## 清单与命令

工作负载叫 `catalog`，容器端口 `8080`。先导出清单，不立刻创建：

```bash
kubectl expose deployment catalog \
  --port=8080 --target-port=8080 \
  --dry-run=client -o yaml > catalog-svc.yaml
```

ClusterIP 形态：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-svc
  labels:
    app: catalog
spec:
  selector:
    app: catalog
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

要从节点外侧访问，改成 NodePort，并固定 30000–32767 之间的端口，方便安全组放行：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-svc
  labels:
    app: catalog
spec:
  selector:
    app: catalog
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 32080
  type: NodePort
```

```bash
kubectl apply -f catalog-svc.yaml
kubectl get svc catalog-svc
kubectl get endpointslices -l kubernetes.io/service-name=catalog-svc
```

`TYPE` 应为 NodePort，浏览器访问任意节点的 `IP:32080`。没有云负载均衡时，也可以在一台有公网地址的机器上做反向代理，把域名转到若干节点的 `32080`；节点增减后要自己改上游。LoadBalancer 类型会由云控制器登记后端，不必手写这份列表。

## 常见问题

- 选择器写错时，ClusterIP 还在，EndpointSlice 为空，表现是超时而不是找不到 Service。
- `port` 是 Service 端口，`targetPort` 是容器端口，写反后连接被拒绝。
- NodePort 在所有节点占用同一端口，不能和别的 Service 冲突。
- 从笔记本访问 ClusterIP 不通是正常的，它没有出集群的路由。
