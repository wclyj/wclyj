# 控制器分类型

Deployment 解决的是“副本可以互换”的服务。下面补三类更具体的控制器：要稳定身份的 StatefulSet、每节点一份的 DaemonSet，以及跑完即停的 Job / CronJob。最后提一下更早的副本控制器，方便读旧清单。

## StatefulSet

适合进程不能随便对调的场景。某个 Pod 换节点时，新实例沿用原序号，并重新挂上原来那块盘。

### 无状态应用

Deployment 管的是可替换副本，默认假设：

- 任意两个 Pod 规格相同，谁接流量都行
- 启动先后没有业务含义
- 落在哪台机器不重要
- 副本数可以随时加减

### 有状态应用

上面几条都不成立时，就要把身份固定下来。

- 每个 Pod 单独编号，不能混用别人的数据目录
- 启动和收缩按序号进行，保证一次只动一个
- 名字、DNS 和卷都跟着序号走
- 典型例子是数据库主从：序号 0 先起来，后面的副本才加入

常见对象包括 MySQL、PostgreSQL，以及靠成员身份做选举的 ZooKeeper、etcd。数据放在外置卷上，控制器只负责把固定序号和固定盘绑在一起。

### 部署有状态应用

先写一个无头 Service：`clusterIP: None`。有头 Service 会做负载均衡，无头 Service 则让每个 Pod 拥有独立 DNS。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ledger-db
  namespace: billing
spec:
  clusterIP: None
  selector:
    app: ledger-db
  ports:
    - name: mysql
      port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ledger-db
  namespace: billing
spec:
  serviceName: ledger-db
  replicas: 3
  selector:
    matchLabels:
      app: ledger-db
  template:
    metadata:
      labels:
        app: ledger-db
    spec:
      containers:
        - name: mysql
          image: mysql:8.0.36
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 8Gi
```

Pod 名是 `ledger-db-0` 这种序号，而不是随机后缀。Service 的 ClusterIP 为 None。稳定地址形如 `ledger-db-0.ledger-db.billing.svc.cluster.local`。Deployment 重建后后缀会变，不能当作永久地址。

## DaemonSet

DaemonSet 按节点铺 Pod，不按业务副本数伸缩。范围内每台机器一份，常用来放日志、指标导出器和存储代理。新节点满足选择器后会自动补一份。

下面这份清单在每个节点放一个日志采集器，把容器日志目录只读挂进去：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-shipper
  namespace: observability
spec:
  selector:
    matchLabels:
      app: log-shipper
  template:
    metadata:
      labels:
        app: log-shipper
    spec:
      containers:
        - name: shipper
          image: elastic/filebeat:8.12.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

进到任意一份，列出 `/var/log`，应能看到节点上的日志：

```bash
kubectl -n observability exec -it log-shipper-7k2mq -- ls /var/log
```

## Job 和 CronJob

- Job：进程成功退出后就结束，不会一直重启成常驻服务
- CronJob：按时间表反复创建 Job

批处理有完成时刻。`completions` 决定要成功几次：默认一次即可，也可以要求 N 次都成功。

### Job

用一个算圆周率的短任务做例子。容器跑完 `pi` 就退出，`.spec.backoffLimit` 限制失败重试次数。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-sample
  namespace: batch
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pi
          image: perl:5.38
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(40)"]
```

```bash
kubectl -n batch get jobs
kubectl -n batch logs job/pi-sample
```

`COMPLETIONS` 变成 `1/1` 后，日志里会留下一串小数。Pod 仍可查到，但状态是 Completed，不会再被拉起来。

### CronJob

定时任务把 Job 模板和时间表放在一起。`schedule` 使用标准的五段 cron。下面每 8 分钟打印一次时间。

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: stamp
  namespace: batch
spec:
  schedule: "*/8 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: stamp
              image: busybox:1.36
              args: ["/bin/sh", "-c", "date; echo tick"]
```

```bash
kubectl apply -f stamp-cron.yaml
kubectl -n batch get cronjobs
kubectl -n batch logs job/stamp-2912041200
```

每次触发都会多一个带时间戳的 Job。用 `successfulJobsHistoryLimit` 限制保留条数。

## 删除 Service 和 StatefulSet

实验做完后按对象删掉，避免无头 Service 和卷声明残留：

```bash
kubectl -n billing delete svc ledger-db
kubectl -n billing delete statefulset ledger-db
```

只删 StatefulSet 不会自动删掉 `volumeClaimTemplates` 创建的 PVC。需要释放磁盘时再单独删除 PVC。

## ReplicationController

ReplicationController（RC）按标签维持 Pod 个数，少则补、多则删。期望即使是 1，也比被节点带走后不会再生的裸 Pod 稳。它只有等值选择器，适合早期的长期服务。

### ReplicaSet

ReplicaSet，简称 RS，做同样的副本维持，但选择器支持集合运算。日常不单独写 RS，而是把它交给 Deployment：Deployment 负责发布策略，RS 负责某一版模板下的副本数。
