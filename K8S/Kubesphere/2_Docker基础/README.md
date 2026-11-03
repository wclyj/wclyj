# Docker：把运行环境一起交付

## 它解决的三件事

同一份业务代码，在开发机、测试机和生产机上依赖的系统库经常不一致。Docker 把“构建、分发、运行”收成同一套动作。

构建。Java、C++ 或前端产物不再直接丢一个安装包到服务器，而是写成镜像。镜像里已经包含运行时和依赖，构建命令是 `docker build`。

分发。镜像放到仓库，公共的是 Docker Hub，企业里则是私有仓库。使用者按名字和标签拉取，和在应用商店选版本类似。

运行。任何装了引擎的机器都执行 `docker run`，不需要再手工安装语言运行时。容器化由此成为交付的默认形态。

### 和虚拟机怎么比

以 VMware 这类虚拟化为参照：每台虚拟机自带完整客户机系统，镜像常常是几个 GB，启动要等系统引导，隔离强，但拷贝和分享都重。

容器共用宿主机内核，镜像可以只有几十到几百 MB，创建就是启动一个被隔离的进程，秒级进入运行。隔离靠内核特性而不是再虚拟化一套硬件，因此比虚拟机轻，也更方便推到仓库给别人拉取。

### 隔离了什么

- CPU 和内存可以限额，避免一个容器占满宿主机。
- 设备访问可以按需放开，默认看不到宿主机磁盘。
- 网络命名空间独立，容器有自己的接口和端口。
- 用户命名空间可以把容器内的 root 映射成宿主机上的普通用户。

## 架构里有哪些角色

- Docker Host：安装了引擎的机器。
- dockerd：宿主机上的后台进程，真正管理镜像和容器。
- Client：`docker` 命令或图形界面，通过套接字把请求发给 dockerd。
- Registry：镜像仓库。Docker Hub 是公共的一个。
- Image：带环境和程序的只读模板。
- Container：镜像启动后的可写运行实例。

日常顺序是：装好引擎，在仓库找到镜像，拉到本地，运行，再用日志和进程列表排错。

## 在 CentOS 上安装

其他发行版的步骤见 <https://docs.docker.com/engine/install/> 。

先卸掉发行版旧包，避免和 docker-ce 冲突：

```bash
sudo yum remove -y docker docker-client docker-client-latest \
  docker-common docker-latest docker-latest-logrotate \
  docker-logrotate docker-engine
```

配置可访问的软件源并安装。`docker-ce` 是社区版引擎，`docker-ce-cli` 是命令行，`containerd.io` 是容器生命周期组件。给 Kubernetes 准备节点时固定小版本，不要装到一个未验证的最新版：

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo \
  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install -y docker-ce-24.0.9 docker-ce-cli-24.0.9 containerd.io-1.6.28
sudo systemctl enable --now docker
```

生产向的 daemon 配置里，把 cgroup 驱动设为 systemd，和 kubelet 保持一致，并限制单文件日志大小。镜像加速域名换成你自己的：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": ["https://docker.m.daocloud.io"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
docker info | awk '/Cgroup Driver|Server Version/'
```

## 镜像与容器命令

### 取得镜像

到仓库查找官方镜像。标签不写时等于 `latest`，构建结果会随时间变化，实验要写明标签：

```bash
docker pull httpd:2.4.58-alpine
docker pull redis:7.2.4
docker images
docker rmi redis:7.2.4
```

`docker images` 列出的是本地镜像。删除镜像前要先停掉使用它的容器。

### 启动、停止、删除

把宿主机的 9080 转到容器的 80，容器名 `edge-web`，退出后自动拉起：

```bash
docker run --name edge-web -d --restart=always -p 9080:80 httpd:2.4.58-alpine
docker ps
docker ps -a
docker stop edge-web
docker start edge-web
docker update edge-web --restart=always
docker rm -f edge-web
```

`docker ps` 只看运行中的，`-a` 包含已退出的。`rm` 不能删掉仍在运行的容器，除非加 `-f`。端口写成 `9080:80`，冒号左侧是宿主机。

### 改页面内容

进容器改文件能立刻看到效果，但容器删除后改动就没了：

```bash
docker exec -it edge-web sh
```

httpd 的默认页面在 `/usr/local/apache2/htdocs/index.html`。更稳妥的是把宿主机目录挂进去，页面留在容器外：

```bash
mkdir -p /srv/edge-web
echo 'edge-ok' > /srv/edge-web/index.html
docker run --name edge-web -d --restart=always \
  -p 9080:80 \
  -v /srv/edge-web:/usr/local/apache2/htdocs:ro \
  httpd:2.4.58-alpine
```

之后只改 `/srv/edge-web` 里的文件即可。`:ro` 让容器不能改宿主机目录。浏览器访问 `http://宿主机:9080/` 应看到 `edge-ok`。

### 把改动存成新镜像

若必须把容器里的文件变更为镜像层，用 commit。作者和说明写成你自己的记录，仓库名不要占用他人的命名空间：

```bash
docker commit -a "lab" -m "replace default page" edge-web notice-httpd:1.0.0
```

commit 出来的镜像没有构建步骤，别人无法复查你改了什么。能写 Dockerfile 时就不要靠 commit。

导出、拷贝、在另一台机器导入，适合仓库不通的内网：

```bash
docker save -o notice-httpd-1.0.0.tar notice-httpd:1.0.0
scp notice-httpd-1.0.0.tar root@10.16.8.21:/tmp/
docker load -i /tmp/notice-httpd-1.0.0.tar
```

### 推到仓库

本地名要改成「仓库用户/镜像名:标签」，登录后再推：

```bash
docker tag notice-httpd:1.0.0 exampleuser/notice-httpd:1.0.0
docker login
docker push exampleuser/notice-httpd:1.0.0
docker logout
```

另一台机器 `docker pull exampleuser/notice-httpd:1.0.0` 后即可运行。推送失败时先确认仓库里已经建好同名仓库，以及标签里没有大写字母。

### 日志、配置文件与拷贝

```bash
docker logs --tail 50 edge-web
docker cp edge-web:/usr/local/apache2/conf/httpd.conf /srv/edge-web/httpd.conf
docker cp /srv/edge-web/httpd.conf edge-web:/usr/local/apache2/conf/httpd.conf
```

改配置更常见的做法仍是挂载，而不是 cp 进去后再重启丢失层外的语义。例如同时挂页面和主配置：

```bash
docker run -d -p 9080:80 \
  -v /srv/edge-web/htdocs:/usr/local/apache2/htdocs:ro \
  -v /srv/edge-web/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro \
  --name edge-web-conf \
  httpd:2.4.58-alpine
```

## 把自己的程序打成镜像

可以从 <https://start.spring.io/> 生成一个只返回问候的 Web 项目，本地打成可执行包。

以前的做法是：构建出 jar，上传到服务器，服务器预先装好 JDK，再 `java -jar`。现在每台机器只装引擎，JDK 留在镜像里。

Dockerfile 使用较小的运行时基础镜像，监听 8090，避免和常见的 8080 示例撞车：

```dockerfile
FROM eclipse-temurin:17-jre
LABEL maintainer=lab
WORKDIR /opt/app
COPY target/greeting-api.jar /opt/app/service.jar
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "/opt/app/service.jar"]
```

```bash
docker build -t greeting-api:1.0.0 .
docker run -d -p 8090:8090 --name greeting greeting-api:1.0.0
```

每个服务都在本机编译再上传构建，服务数量多时会很慢。后续可以把编译放进持续集成，镜像只在流水线里产生。

`ENTRYPOINT` 的 exec 格式（JSON 数组）让 `java` 成为一号进程，能收到停止信号。`CMD` 提供默认参数，`docker run` 后面额外写的参数会覆盖 `CMD`，但会追加到 `ENTRYPOINT` 后面。两者至少要有一个，否则容器启动后立即退出。

分享这只镜像：

```bash
docker tag greeting-api:1.0.0 exampleuser/greeting-api:1.0.0
docker push exampleuser/greeting-api:1.0.0
```

另一台机器拉取后，用同样的 `docker run` 即可，不需要再装 JDK。

## 带上一个依赖服务

问候服务若要读写缓存，再起一个 Redis。配置和数据都放在宿主机，避免容器删除后数据消失：

```bash
mkdir -p /srv/session/data
docker run -d --name session \
  -p 6380:6379 \
  -v /srv/session/redis.conf:/etc/redis/redis.conf:ro \
  -v /srv/session/data:/data \
  redis:7.2.4 \
  redis-server /etc/redis/redis.conf
```

`redis.conf` 里把 `dir` 设为 `/data`，并改掉默认口令。应用容器通过宿主机内网地址访问 `6380`，或者让两个容器加入同一用户自定义网络，用容器名 `session` 互访。

## 常见问题

- 端口写成 `80:9080`，方向反了，容器里并没有进程听 9080。
- 挂载目录盖住了镜像里的原有文件，页面变成空目录的列表。先 `docker cp` 出默认内容再挂载。
- `latest` 在不同时间拉取到不同层，排障时无法复现，部署清单里写死标签。
- commit 出来的镜像含有不该存在的口令文件，推仓库之前先检查层里的内容。
