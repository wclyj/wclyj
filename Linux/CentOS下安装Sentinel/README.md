# CentOS 下安装 Sentinel

## 前言

Sentinel 控制台是一个可执行 jar，用来查看限流和熔断。下面只写在 CentOS 上把它跑起来。

## 下载 Sentinel

发布页：https://github.com/alibaba/Sentinel/releases

下载与应用客户端主版本一致的 dashboard jar，例如 `sentinel-dashboard-1.8.8.jar`，放到 `/opt/flowguard`。控制台端口改成 `8719`，避免和别的管理端撞车。

启动脚本 `start.sh`：

```bash
#!/bin/bash
cd /opt/flowguard
nohup java -Xms128m -Xmx256m \
  -jar sentinel-dashboard-1.8.8.jar \
  --server.port=8719 \
  > /var/log/flowguard.out 2>&1 &
echo $! > /run/flowguard.pid
```

关闭脚本 `stop.sh`：

```bash
#!/bin/bash
pid=$(pgrep -f sentinel-dashboard-1.8.8.jar)
if [ -z "$pid" ]; then
  echo "flowguard is not running"
else
  echo "stopping $pid"
  kill "$pid"
fi
```

赋予执行权限后启动：

```bash
chmod 755 /opt/flowguard/start.sh /opt/flowguard/stop.sh
/opt/flowguard/start.sh
```

浏览器打开 `http://10.8.20.18:8719`。默认用户名和口令都是 `sentinel`，登录后应先改掉。左侧还没有应用是正常的，要等业务进程连上控制台并开始上报，机器列表才会出现。
