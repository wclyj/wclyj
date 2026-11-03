# 一次日志写满容器磁盘的排查

## 前言

订单服务页面大面积打不开。准备先看 `order-api` 的日志：

```bash
tail -n 50 /var/log/order-api/app.log
```

命令卡住，几乎不再刷出新行。当时以为是进程把终端堵住了，想先把现场做成镜像再慢慢看：

```bash
docker commit --message "before cleanup" 6b21c0aa91 shop/order-api:frozen
```

引擎马上返回：

```text
Error response from daemon: mkdir /var/lib/docker/overlay2/91ab30c0e77: no space left on device
```

磁盘没有空间时，连提交新层需要的目录都建不出来。于是打算删掉不用的镜像：

```bash
docker image ls
```

删除同样失败，报的还是空间不足。问题就不一定在镜像层上。先看挂载点：

```bash
df -hP
```

```text
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        1.8G     0  1.8G   0% /dev
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           1.9G  210M  1.7G  12% /run
/dev/vda1        80G   80G     0 100% /
overlay          80G   80G     0 100% /var/lib/docker/overlay2/55ac0912/merged
shm              64M     0   64M   0% /var/lib/docker/containers/6b21c0aa91/shm
```

80 GB 的根分区已经 100%。镜像删不掉，是因为删除本身也要写元数据。接着区分镜像和容器各自占了多少：

```bash
docker system df -v | head -n 20
```

示意：

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          6         4         4.2GB     1.1GB
Containers      5         5         63.4GB    0B
Local Volumes   2         2         800MB     0B
Build Cache     0         0         0B        0B
```

容器一项远大于镜像。某一个容器的可写层就有六十多 GB。进入该容器，按目录往下看（宿主机上也可以对 overlay 上层目录做同样的事）：

```bash
du -h -d 1 /
```

`/opt` 约占 61 GB。再收窄：

```bash
du -h -d 1 /opt/order-api
```

`/opt/order-api/logs` 独占大约 60 GB。打开最新的日志，内容是同一条空指针在循环里刷屏，文件因此以很快的速度长大。

处理顺序是先让进程停下错误输出，再删文件。若只删文件而循环还在，句柄仍会把空间占住：

```bash
# 在确认进程已停写之后
rm -f /opt/order-api/logs/app.log
df -h /
```

容器里的使用率掉下去之后，回到宿主机再看 `df -h`，可用空间回到数十 GB。重新启动订单服务，`tail` 可以正常跟上新日志。

事后应把日志改成滚动（按大小或按天切分，并限制保留份数），同时修掉那个空指针。容器磁盘和宿主机根分区是同一块盘时，一个服务的日志就能把整机写满，提交镜像、拉取新层都会一起失败。
