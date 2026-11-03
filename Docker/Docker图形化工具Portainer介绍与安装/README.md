# Docker 图形界面 Portainer：介绍与安装

## 介绍

Portainer 是一个轻量的图形管理端，用来查看和操作 Docker 环境。它自己也跑在容器里，可以管理容器、镜像、卷和网络，既能接单机引擎，也能接 Swarm。中小规模的主机用它做日常查看和启停通常够用。

官方提供过公开演示环境，地址和口令会变动，以官网当前说明为准，不要把演示口令写进自己的环境。

## Docker 方式安装

先准备一块数据卷，让设置在容器重建后还在：

```bash
docker volume create edge-ui-data
```

再启动管理容器。当前镜像同时提供 HTTP 与 HTTPS，练习时可以只发布 HTTPS 端口：

```bash
docker run -d \
  --name edge-ui \
  --restart unless-stopped \
  -p 19443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v edge-ui-data:/data \
  portainer/portainer-ce:2.21.4
```

浏览器打开 `https://主机:19443`。首次进入要设置管理员口令。口令只存在这块数据卷里，卷被删掉就要重新初始化。

接下来选择要接管哪一台引擎：

- Local：本机，通过挂载进来的套接字
- Remote：另一台已开放管理接口的引擎
- Agent：在目标机上跑的 Portainer Agent
- Azure：托管的容器实例

单机练习选 Local 即可。

## Docker Compose 方式安装

主机上需要已经有 Compose 插件。

```bash
mkdir -p /srv/edge-ui && cd /srv/edge-ui
```

编写 `edge-ui.yaml`：

```yaml
services:
  edge-ui:
    image: portainer/portainer-ce:2.21.4
    container_name: edge-ui
    restart: unless-stopped
    ports:
      - "19443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
```

启动：

```bash
docker compose -f edge-ui.yaml up -d
```

访问 `https://主机:19443`，第一次登录设置管理员口令。进去之后，本机容器、镜像和卷会列在同一页。界面语言可以在账号设置里切换，不必再额外挂一套第三方语言包。

## 使用 Portainer 管理其它主机

管本机只需要套接字。要管另一台机器，优先在那台机器上跑 Agent，由 Agent 主动连到管理端，避免把 Docker 的明文 API 暴露到网络上。

在被管主机上：

```bash
docker run -d \
  --name edge-agent \
  --restart unless-stopped \
  -p 19001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:2.21.4
```

回到管理端，打开“环境 / Endpoints”，添加 Agent 环境，地址填 `被管主机:19001`。添加成功后，首页会出现两台环境，点进去再操作对应主机上的容器。

有些旧文档会改 `daemon.json`，让引擎监听 `tcp://0.0.0.0:2375`。这个端口没有认证，扫描到就会被拿去跑别人的容器。练习环境也不要打开它。若必须远程调用 API，应放在内网并配 TLS，或继续用 Agent。

## 使用 Portainer 部署 Nginx 服务

环境多于一台时，先在首页选中目标主机，再进行拉取和创建。下面假设选中的是 `10.8.20.24`。

1. 打开 Images，名称填 `nginx:1.27-alpine`，执行拉取。
2. 拉取完成后进入 Containers，用该镜像创建容器。名称写 `web-lab`，端口映射填宿主机 `18080` 对容器 `80`，重启策略选 unless-stopped。
3. 创建成功后，容器列表里能看到 `18080->80`。

浏览器访问：

```text
http://10.8.20.24:18080
```

页面出现 Nginx 的欢迎信息，说明发布端口和进程都正常。随后可以在同一界面挂卷、改环境变量或看日志，不必回到命令行。
