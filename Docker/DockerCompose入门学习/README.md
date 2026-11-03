# Docker Compose 入门

## 简介

只用 Dockerfile 配合构建和运行时，一次只能处理一个容器。服务数量上来之后，端口、数据卷和启动先后都要手工记住。Compose 把这一组容器写进同一份 YAML，用一条命令创建或停掉整组服务。

### 官方介绍

- 声明并运行多个容器
- 用 YAML 描述运行环境

Compose 把多容器应用的定义和启动收成一套流程。镜像如何构建、端口如何映射、服务依赖谁，都放在同一个文件里。执行一次启动命令，网络、卷和容器会按文件建出来。开发机、流水线和预发环境可以共用这份定义。

常见用法分三步：

- 用 Dockerfile 固定程序及其依赖，换机器也能重建
- 在 Compose 文件里写出各个服务怎样拼在一起
- 执行启动命令，整组应用一起跑起来

### 作用

批量把容器编排起来。

它是 Docker 维护的编排工具，需要单独安装。Dockerfile 解决“这个程序换一台机器还能跑”；Compose 解决“站点、缓存、数据库要一起跑”。

一份精简文件可以写成：

```yaml
services:
  shop:
    build: .
    ports:
      - "8088:8088"
    volumes:
      - ./src:/srv/shop
      - shoplog:/srv/log
    depends_on:
      - cache
  cache:
    image: redis:7.2-alpine
volumes:
  shoplog: {}
```

服务再多，仍然是一条启动命令。

两个常用名词：

- service：一个应用角色，例如站点、缓存、数据库，背后是一个或多个容器
- project：这一组互相协作的 service

## 常见的 Docker Compose 脚本

公开仓库里有不少现成文件，部署开源组件时可以先找一份，再改镜像标签、端口和数据目录。拿到手先核对口令和卷挂载，练习用的口令不要带到生产。

## 安装 Docker Compose

安装步骤以 Docker 文档中的 Compose 插件说明为准。新环境推荐 CLI 插件，命令写成 `docker compose`（中间是空格）。老机器上仍常见独立二进制 `docker-compose`。

### 下载

先确认引擎可用：

```bash
dnf -y install docker-ce
docker version
systemctl enable --now docker
```

再放一份独立二进制，版本按当时的发布页选择，下面以 v2.27.1 为例：

```bash
sudo curl -fL "https://github.com/docker/compose/releases/download/v2.27.1/docker-compose-linux-$(uname -m)" \
  -o /opt/bin/docker-compose
```

### 授权

```bash
cd /opt/bin
sudo chmod 755 docker-compose
docker-compose version
```

能打出版本号，就说明这个文件可以执行。

### 创建文件体验

下面做一个访问计数小站。页面用 Python 的 Bottle，计数放在 Redis。本机不用预装这两样，镜像里自带。

```bash
mkdir -p ~/labs/tally && cd ~/labs/tally
```

`server.py`：

```python
import time
import redis
from bottle import route, run

cache = redis.Redis(host="cache", port=6379, socket_timeout=2)

def next_count():
    left = 4
    while True:
        try:
            return int(cache.incr("tally"))
        except redis.exceptions.ConnectionError:
            left -= 1
            if left < 0:
                raise
            time.sleep(0.3)

@route("/")
def index():
    return "tally=%s\n" % next_count()

if __name__ == "__main__":
    run(host="0.0.0.0", port=8088)
```

`requirements.txt`：

```text
bottle==0.12.25
redis==5.0.4
```

### 创建 Dockerfile

```dockerfile
FROM python:3.12-alpine
WORKDIR /srv/tally
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
EXPOSE 8088
CMD ["python", "server.py"]
```

这段文件告诉构建器：以 Python 3.12 的 Alpine 变体为底，工作目录定为 `/srv/tally`，先装依赖再拷入脚本，进程监听 8088。

### 定义服务在 Compose 文件中

项目目录里放 `compose.yaml`：

```yaml
services:
  shop:
    build: .
    ports:
      - "8088:8088"
  cache:
    image: redis:7.2-alpine
```

文件定义了 shop 和 cache。shop 用当前目录的 Dockerfile 构建，并把容器的 8088 映到宿主机的 8088。cache 直接用仓库里的 Redis 镜像，不对外发布端口。

#### 站点服务

构建上下文是当前目录。容器端口与 Bottle 的监听端口对齐。只想改宿主机端口时，改冒号左边的数字。

#### 缓存服务

镜像从公共仓库拉取。同一项目网络里，服务名 `cache` 就是 DNS 名，所以脚本里写 `host="cache"` 即可连上，不必写容器 IP。

### 把计数服务拉起来

进入 `~/labs/tally` 后执行：

```bash
docker compose up
```

终端输出大致如下（名称和时间仅作示意）：

```text
Network tally_default          Created
Container tally-cache-1        Created
Container tally-shop-1         Created
Attaching to cache-1, shop-1
cache-1  | Ready to accept connections
shop-1   | Bottle v0.12.25 server starting up
shop-1   | Listening on http://0.0.0.0:8088/
```

另开终端检查：

```bash
curl -s localhost:8088
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.Size}}'
```

第一次访问应返回 `tally=1`。镜像列表里能看到 Redis，以及本次构建出来的站点镜像。

### 网络规则

```bash
docker network ls
docker network inspect tally_default
```

Compose 会为项目建一张桥接网络。inspect 结果的容器列表里应同时出现 shop 和 cache。它们用服务名互通，不依赖写死的地址。

### 关闭 docker compose

前台运行时按 Ctrl+C 结束。若要连项目网络一起清掉：

```bash
docker compose down
```

### 总结

上面的练习可以收成几步：

- 编写应用脚本
- 用 Dockerfile 把应用打成镜像
- 用 YAML 声明站点与缓存
- 执行 `docker compose up`
- 引擎先建网络，再按文件构建或拉取，最后启动容器

没有 Compose 时，每个镜像都要单独 `docker run`，网络和名字也要自己管。写成 YAML 之后，启动和停止都收成一条命令。

## yaml 规则

文件大体分三层：可选的版本号、services，以及 volumes、networks、configs 这类顶层对象。

```yaml
services:
  cache:
    image: redis:7.2
  edge:
    image: nginx:1.27-alpine
  shop:
    build: ./shop
volumes: {}
networks: {}
configs: {}
```

带配置对象的例子：

```yaml
services:
  cache:
    image: redis:7.2
    configs:
      - source: cache_note
        target: /usr/local/etc/redis/note.txt
configs:
  cache_note:
    file: ./note.txt
```

### 依赖关系

站点应等数据库和缓存的容器先被创建。`depends_on` 只约束启动顺序，不等待端口真正可连。需要等健康检查通过时，再写 `condition: service_healthy`。

```yaml
services:
  shop:
    build: .
    depends_on:
      - pg
      - cache
  cache:
    image: redis:7.2-alpine
  pg:
    image: postgres:16-alpine
```

## 快速搭建 WordPress

文档站有官方示例。下面改了口令、端口和卷名，只适合本机练习。

```bash
mkdir -p ~/labs/press && cd ~/labs/press
```

`compose.yaml`：

```yaml
services:
  db:
    image: mysql:8.0
    volumes:
      - pressdb:/var/lib/mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root-lab-7730
      MYSQL_DATABASE: press
      MYSQL_USER: presser
      MYSQL_PASSWORD: press-lab-2208
  cms:
    image: wordpress:6.5-apache
    depends_on:
      - db
    ports:
      - "18080:80"
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: presser
      WORDPRESS_DB_PASSWORD: press-lab-2208
      WORDPRESS_DB_NAME: press
volumes:
  pressdb: {}
```

放到后台：

```bash
docker compose up -d
```

浏览器打开 `http://主机:18080` 进入安装页。真实项目往往还要用 Dockerfile 打出自己的程序包，文件齐了再启动。构建有改动时加上 `--build`。

排障时优先看 `docker compose logs`，再看 `docker compose ps` 里哪个服务没有起来。

## docker-compose 搭建微服务

一组后端服务可以按这个顺序接上：

- 每个服务先在本地单独跑通
- 需要定制运行环境的服务各写一份 Dockerfile
- 用一份 Compose 文件串起网关、业务服务和中间件
- 目录放到主机后执行 `docker compose up -d`
- 代码或 Dockerfile 变了，就加 `--build` 重建

## Docker 小结

- 镜像通过 run 变成容器
- Dockerfile 负责把服务打包成镜像
- Compose 负责把多个服务一起拉起
- 同一项目网络里用服务名互相访问

## 参考

- https://docs.docker.com/compose/
