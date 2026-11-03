# 用 Rancher 2 接管集群

Rancher 2 的中文手册按版本分册。下面用容器把控制台跑起来，再把一套已经能用的 Kubernetes 接进去，而不是在向导里从零创建节点。

## 安装 Rancher

在一台不承担业务 Pod 的机器上启动。数据目录挂到宿主机，避免容器删除后登录状态丢失。示例使用 2.7.6，宿主机地址 `10.20.8.40`。

```bash
docker run -d --name ranch-console --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /var/lib/ranch-console:/var/lib/rancher \
  rancher/rancher:v2.7.6
docker ps --filter name=ranch-console
docker logs -f ranch-console
```

日志里出现监听 80 与 443 的提示后，浏览器打开 `https://10.20.8.40/`。首次进入要设置管理员口令，并确认访问地址。口令不要沿用示例里的占位符。完成后落在集群列表页。

## 导入已有集群

控制台里选择添加集群，再选“导入已有 Kubernetes”。给集群起名 `edge-lab`，页面会生成一段只针对这次导入的 kubectl 命令，以及一份可下载的清单。

在目标集群的控制面节点执行页面给出的命令。若控制面访问不了 Rancher 的证书域名，先把清单保存为 `edge-import.yaml`，再应用：

```bash
kubectl apply -f edge-import.yaml
```

旧集群若仍拒绝清单，多半是里面残留了 `extensions/v1beta1` 的 Deployment。把 `apiVersion` 改成 `apps/v1`，并在 `spec` 下补上与模板标签一致的 `selector.matchLabels`，然后重新应用。

导入会在 `cattle-system` 命名空间创建代理 Pod：

```bash
kubectl get pods -n cattle-system
```

看到 cattle 相关 Pod 为 Running 后，回到控制台点完成。列表里的 `edge-lab` 变为可用，即可在界面上查看节点和项目，不必再把 kubeconfig 发给每个操作者。代理只是多了一条管理通道，原来的 kubeadm 或二进制控制面仍然负责调度。
