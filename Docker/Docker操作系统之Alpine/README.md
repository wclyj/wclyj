# Docker 里的 Alpine 系统

## 前言

带 `alpine` 标签的镜像通常比同软件的默认标签小一截，例如 `eclipse-temurin:17-jre-alpine` 和 `nginx:1.27-alpine`。这些镜像底下都站着一个 Linux 发行版。常见的是 Ubuntu、CentOS、Debian，Alpine 是另一条产品线，安装包和 C 库都更精简，所以叠在它上面的应用镜像也更小。

一组示意体积（不同时间的仓库数字会变，只用来对比量级）：

```text
REPOSITORY   TAG      IMAGE ID       SIZE
alpine       3.20     a1c0e9b74d11   7.4MB
debian       12       b20d88c31e55   117MB
ubuntu       24.04    c88a17f0de44   78MB
rockylinux   9        d31e6aa09e02   160MB
```

Alpine 本身只有数兆，做出来的应用镜像通常也会跟着变小。

## Alpine 操作系统

Alpine 是面向安全场景的小型 Linux。它用 musl 代替 glibc，用 BusyBox 提供基础命令，从而压低磁盘和内存占用；包数量又比纯 BusyBox 完整，所以适合当容器底包。包管理器是 `apk`，对应 CentOS 的 `yum`/`dnf` 和 Ubuntu 的 `apt`。可安装的包在 https://pkgs.alpinelinux.org/packages 查询，装一个编辑器就是：

```bash
apk add --no-cache vim
```

`--no-cache` 会在安装后丢掉索引缓存，避免这一层白占空间。

Docker Hub 上的官方 Alpine 仓库（https://hub.docker.com/_/alpine）提供这套几兆字节的根文件系统：

```bash
docker pull alpine:3.20
```

官方说明里常用数据库镜像做对比：基于 Alpine 的变体可以只有几十兆，基于 Ubuntu 的变体往往过百兆，差出数倍。因此不少官方镜像把 Alpine 当作推荐的瘦身底包。带来的好处是拉取更快、攻击面更小、在主机之间搬动更轻、运行时内存更省。

## 使用 Alpine 镜像

本地没有镜像时，下面的命令会先拉取再运行，并打印一行字：

```bash
docker run --rm alpine:3.20 echo "hi from alpine"
```

示意输出：

```text
Unable to find image 'alpine:3.20' locally
3.20: Pulling from library/alpine
Digest: sha256:example
Status: Downloaded newer image for alpine:3.20
hi from alpine
```

`--rm` 让容器退出后自动删除，适合这种一次性命令。

## 迁移至 Alpine

多数官方镜像已经提供 Alpine 变体，改标签就能换底包：

```text
ubuntu / debian  ->  alpine:3.20
python:3.12      ->  python:3.12-alpine
ruby:3.3         ->  ruby:3.3-alpine
nginx:1.27       ->  nginx:1.27-alpine
```

在 Alpine 上追加软件，统一用：

```bash
apk add --no-cache <包名>
```

编译型依赖（头文件、编译器）装上之后，如果运行时不再需要，应在同一层里删掉，否则镜像会重新变大。

## 思考

体积小、官方又提供现成标签，是否意味着 Alpine 可以取代 CentOS、Ubuntu 和 Debian？不能直接这么推断，原因有三条：

- 在 Alpine 上写 Dockerfile 没有在 Debian 系上顺手。包名不同，有的软件根本没有 apk 包，得自己编译。
- 单个镜像确实更小，但多个镜像若共用同一基础层，Docker 的叠加文件系统不会把这一层再存一份。此时换 Alpine 省下的是“每一层的差异”，不是把所有容器的磁盘各减一遍。
- 依赖 glibc 的程序可能无法在 musl 上运行，或者要换官方没有提供的构建方式。遇到这类软件，应留在 glibc 发行版上。

选型时先确认依赖能在 musl 上工作，再比较拉取时间和运行内存，而不是只看 `docker image ls` 里的 SIZE 列。

## 参考

- Alpine 项目：https://alpinelinux.org/
- 发行版源码：https://github.com/alpinelinux
- 官方镜像说明：https://hub.docker.com/_/alpine/
