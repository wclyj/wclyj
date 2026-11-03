# 把镜像推到云端私有仓库

## 前言

公共仓库在境外时，拉取和推送经常超时。可以在云厂商的镜像产品里建私有仓库，把常用镜像先放进去，同一地域的主机再拉会快很多。下面用示例服务 `ledger-api` 走通登录、打标签和推送。

## 阿里云容器镜像服务

开通入口在容器镜像服务控制台。创建命名空间，例如 `team-lab`，再在该空间下建仓库 `ledger-api`。访问凭证页面设置固定密码，并复制登录命令。密码和账号不要写进仓库。

在已经装好引擎的主机上登录，地域换成自己控制台里的地址：

```bash
docker login --username='dev-user@example.com' registry.cn-hangzhou.aliyuncs.com
```

登录成功后，凭证写在当前用户的 docker 配置里。

## 提交镜像

先看本地已有镜像：

```bash
docker image ls
```

示意：

```text
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
ledger-api    1.4       7c2a91e0b418   2 hours ago    186MB
```

给已有镜像加远程标签。格式是“本地引用 + 仓库地址/命名空间/仓库名:标签”：

```bash
docker tag 7c2a91e0b418 registry.cn-hangzhou.aliyuncs.com/team-lab/ledger-api:1.4.0
```

镜像还只是一个容器、没有现成镜像时，可以先提交再打标签。提交会把容器可写层固化，适合临时留档，不适合代替 Dockerfile：

```bash
docker ps -a --format '{{.ID}} {{.Names}}'
docker commit --message "snapshot before upgrade" a91c0e77b2 ledger-local:snap
docker tag ledger-local:snap registry.cn-hangzhou.aliyuncs.com/team-lab/ledger-api:snap
```

打完标签再列一次镜像，同一 IMAGE ID 下会多出远程仓库那一行。

推送：

```bash
docker push registry.cn-hangzhou.aliyuncs.com/team-lab/ledger-api:1.4.0
```

示意输出：

```text
1.4.0: digest: sha256:9f0c2e8a1b77example size: 2411
```

回到控制台的仓库页，标签列表里应出现 `1.4.0`。其他主机登录同一账号后，用完整仓库地址拉取即可，不必再访问境外公共仓库。
