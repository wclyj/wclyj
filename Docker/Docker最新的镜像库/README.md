# 更换镜像站

## 安装 docker

```bash
dnf -y install docker-ce
```

## 修改配置文件

编辑 `/etc/docker/daemon.json`，写入镜像站后执行 `systemctl restart docker`。

```json
{"registry-mirrors":["https://docker.m.daocloud.io"]}
```
