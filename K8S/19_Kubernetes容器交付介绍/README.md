# 容器交付

这一章走通一件具体的事：把一个会监听 HTTP 的 Java 服务做成镜像，推进仓库，再在集群里拉起来。

## 容器交付流程

- 开发
  - 写业务代码和测试
  - 写 Dockerfile，声明用什么基础镜像、怎么启动
- 集成
  - 编译并打出构建产物
  - 构建镜像
  - 推到仓库
- 部署
  - 命名空间、镜像拉取凭据和配额先备好
  - 用工作负载描述副本
  - 用 Service 给出稳定端口
  - 需要域名和路径转发时再加 Ingress
- 运行中
  - 看探针、日志和指标
  - 根据事件排查拉镜像失败或崩溃重启
  - 改镜像标签做滚动升级

## 在集群里发布 Java 服务的顺序

1. 用 Dockerfile 把可执行包封进镜像
2. 推到团队使用的仓库
3. 用 Deployment 声明副本和标签
4. 用 Service 或 Ingress 把端口露出去
5. 按指标和版本号做后续变更

## 实操

### 准备构建产物

示例项目 `ledger-api` 打成可执行包，监听 9094，`/health` 返回一行文本。Maven 工程在仓库根目录执行：

```bash
mvn -q -DskipTests package
```

成功后得到 `target/ledger-api.jar`。JDK 与 Maven 装在构建机上即可，不必装进最终运行的业务镜像以外的地方。

### 编写 Dockerfile

基础镜像只保留运行期，构建期的 Maven 不进最终层。

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /opt/app
COPY target/ledger-api.jar app.jar
EXPOSE 9094
ENTRYPOINT ["java","-jar","/opt/app/app.jar"]
```

`ENTRYPOINT` 用数组形式，避免再套一层 shell 把 Java 收成僵尸进程。不要在参数末尾加 `&`，容器里的主进程必须留在前台。

### 构建镜像

把工程目录拷到构建机，在含 Dockerfile 的目录执行：

```bash
docker build -t ledger-api:2.3.1 .
docker images ledger-api
```

列表里应出现 `ledger-api`，标签 `2.3.1`。

### 先在单机验证

集群会放大配置错误。先用本机端口确认进程能起来：

```bash
docker run -d --name ledger-try -p 19094:9094 ledger-api:2.3.1
curl -fsS http://127.0.0.1:19094/health
```

返回预期正文后再停掉试运行容器。若这里就失败，不要继续推镜像。

### 推送镜像

在仓库里创建项目 `team-pay/ledger-api`，记下登录地址。下面用 `registry.example.net` 代替真实仓库，账号按现场替换。

```bash
docker login registry.example.net
docker tag ledger-api:2.3.1 registry.example.net/team-pay/ledger-api:2.3.1
docker push registry.example.net/team-pay/ledger-api:2.3.1
```

推送完成后，在仓库页面确认标签 `2.3.1` 的摘要已经存在。私有仓库还要在集群里准备 `imagePullSecret`，并写进 Pod 的 `imagePullSecrets`。

### 用控制器运行

先渲染一份 Deployment 骨架，镜像写完整地址，避免节点去公共仓库找同名镜像。

```bash
kubectl -n shop create deployment ledger \
  --image=registry.example.net/team-pay/ledger-api:2.3.1 \
  --dry-run=client -o yaml > ledger.yaml
```

把容器端口和探针补进骨架后再提交。精简后的关键字段如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ledger
  namespace: shop
  labels:
    app: ledger
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ledger
  template:
    metadata:
      labels:
        app: ledger
    spec:
      containers:
        - name: ledger
          image: registry.example.net/team-pay/ledger-api:2.3.1
          ports:
            - containerPort: 9094
          readinessProbe:
            httpGet:
              path: /health
              port: 9094
```

```bash
kubectl apply -f ledger.yaml
kubectl -n shop get pods -l app=ledger
kubectl -n shop scale deployment ledger --replicas=4
```

四个 Pod 都 Ready 后，再暴露端口。`--port` 是 Service 端口，`--target-port` 是容器端口，两者可以不同，这里保持 9094。

```bash
kubectl -n shop expose deployment ledger --port=9094 --target-port=9094 --type=NodePort --name=ledger-np
kubectl -n shop get svc ledger-np
```

假设分配到节点端口 31847，集群内用 ClusterIP，集群外用任意节点 IP 加节点端口：

```bash
curl -fsS http://10.96.12.40:9094/health
curl -fsS http://10.20.8.21:31847/health
```

两条都应返回与单机试运行相同的正文。

### 运行期注意

镜像标签要前进，例如下一次发 `2.3.2`，不要反复覆盖同一个 `latest` 而不改模板。升级用 `kubectl -n shop set image deployment/ledger ledger=registry.example.net/team-pay/ledger-api:2.3.2`，确认就绪后再观察错误率。需要回退时走 Deployment 的修订记录，而不是手工删 Pod。
