# 集群资源监控

## 概述

### 要看哪些数

只看机器通不通，不够判断容量。至少分开两层：

- 集群
  - 各节点的 CPU、内存、磁盘和网络
  - 节点是否 Ready、是否被打上污点
  - 当前 Pod 数量是否顶到每节点上限
- 工作负载
  - 容器的实际用量相对 requests 和 limits 的比例
  - 进程自己暴露的业务指标，例如请求耗时

### 用什么搭

实验组合是 Prometheus 加 Grafana。

- Prometheus 按固定间隔拉取目标的 HTTP 指标，本身也带时间序列库和告警规则。接入方只要提供文本格式的指标页，不必先装一套代理框架。
- Grafana 负责查询和画图，数据源不限于 Prometheus。

数据流是：节点上的导出器暴露指标，Prometheus 抓取后存下来，Grafana 只读这些数据。告警规则可以后加，本章先把图做出来。

## 部署

### 节点导出器

每个工作节点跑一份导出器，监听本机指标。镜像标签和端口按实验固定，避免和已有的 9100 冲突。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: host-metrics
  namespace: observability
  labels:
    app: host-metrics
spec:
  selector:
    matchLabels:
      app: host-metrics
  template:
    metadata:
      labels:
        app: host-metrics
    spec:
      containers:
        - name: exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
          ports:
            - name: http
              containerPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
---
apiVersion: v1
kind: Service
metadata:
  name: host-metrics
  namespace: observability
  labels:
    app: host-metrics
spec:
  type: NodePort
  selector:
    app: host-metrics
  ports:
    - name: http
      port: 9100
      targetPort: http
      nodePort: 31721
```

```bash
kubectl create ns observability
kubectl apply -f host-metrics.yaml
kubectl -n observability get ds,svc host-metrics
```

若清单是从旧文档抄来的，常见报错是 `no matches for kind "DaemonSet" in version "extensions/v1beta1"`。那个 API 已经移除。把 `apiVersion` 改成 `apps/v1`，并补上 `selector.matchLabels`，标签要和模板一致，否则 1.16 之后的集群直接拒绝。

浏览器访问 `http://<节点IP>:31721/metrics`，应看到以 `node_` 开头的文本。

### Prometheus

至少四份对象，都放在 `observability`：

- ConfigMap：抓取间隔和目标地址
- Deployment：Prometheus 进程，限制内存，挂上上述配置
- Service：给 Grafana 和一个节点端口使用
- RBAC：允许它列出节点和 Pod，否则服务发现是空的

先给权限，再给配置，最后起进程。顺序反了时，Pod 会启动但抓取列表为空。

```bash
kubectl apply -f prom-rbac.yaml
kubectl apply -f prom-config.yaml
kubectl apply -f prom-deploy.yaml
kubectl apply -f prom-svc.yaml
kubectl -n observability get pods,svc -l app=prom
```

ConfigMap 里静态抓取节点导出器时，地址写 Service DNS：

```yaml
scrape_configs:
  - job_name: host-metrics
    static_configs:
      - targets: ["host-metrics.observability.svc:9100"]
```

假设 Service 使用 NodePort `31790`。打开 `http://10.20.8.21:31790`，在 Status、Targets 里应看到 `host-metrics` 为 UP。

### Grafana

旧清单同样可能写着 `extensions/v1beta1`。改成 `apps/v1`，并加上选择器：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: board
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: board
  template:
    metadata:
      labels:
        app: board
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:10.4.2
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "lab-board-2026"
```

```bash
kubectl apply -f board-deploy.yaml
kubectl apply -f board-svc.yaml
```

若还要用域名转发，再提交 Ingress；实验阶段 NodePort 足够。Service 把 3000 映射为节点端口 `31826`。

```bash
kubectl -n observability get svc board
```

打开 `http://10.20.8.21:31826`，用户名 `admin`，口令是上面设置的 `lab-board-2026`。不要使用空口令或把口令写进镜像。

### 添加数据源

进入 Connections 或 Data sources，类型选 Prometheus。地址填集群内 DNS，而不是节点端口，这样面板和 Prometheus 走同一网络：

```text
http://prom.observability.svc:9090
```

保存前点测试连接。把数据源命名为 `lab-prom`，后面的面板只绑定这个名字。

### 导入面板

Dashboards 里选择 Import，填入社区面板编号 `1860`（Node Exporter 常用总览）。导入向导的数据源下拉框选 `lab-prom`。

导入后应能看到各节点 CPU、内存和磁盘。若面板是空的，回到 Prometheus 的 Targets 页看抓取是否失败，而不是先换一个面板编号。业务指标要另加应用的抓取任务，节点面板不会自动出现请求耗时。
