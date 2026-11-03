# kubeadm 与二进制安装对照

两条路得到的都是同一套控制面。差别在于谁生成证书、谁写 systemd 单元，以及出问题后能不能直接看到参数。

## 用 kubeadm 搭

1. 准备至少三台虚拟机，做时间同步、关 swap、放行内网。
2. 每台安装容器运行时、kubeadm、kubelet、kubectl。组件指定同一小版本，例如 `1.28.8`，避免控制面和工作节点版本漂移。
3. 只在第一台控制面执行 `kubeadm init`。镜像仓库在境外时，加上 `--image-repository` 指向已同步的仓库。
4. 安装 CNI。未安装前节点停在 NotReady，例如 `kubectl apply -f calico.yaml`。
5. 把 init 打印的 `kubeadm join` 拿到工作节点执行。
6. 用一个简单工作负载验收，确认 Pod 能跨节点拿到地址并被 Service 访问。

kubeadm 会把 apiserver、控制器管理器、调度器和 etcd 写成静态 Pod，不用手写单元文件。

## 用二进制包搭

1. 系统初始化与上一条相同。
2. 用 cfssl 自签证书：集群 CA（`ca.pem` 与私钥）、etcd 对等证书、apiserver 服务证书。SAN 要包含控制面 IP、`kubernetes` 这个服务名，以及将来的负载均衡地址。
3. 部署 etcd。二进制和证书放到固定目录，写成 systemd 服务并设为开机启动。三成员健康后再动 Kubernetes。
4. 部署 kube-apiserver、kube-controller-manager、kube-scheduler，各自一个单元，证书路径和 etcd 地址手写。
5. 部署容器运行时、kubelet、kube-proxy。kubelet 用引导令牌申请客户端证书，控制面批准后节点才变为 Ready。
6. 安装 CNI，再跑验收工作负载。

这里 etcd 和四个控制面进程都是普通 systemd 服务，日志用 `journalctl -u` 查看。升级就是替换二进制并滚动重启，不经过 `kubeadm upgrade`。

## 怎么选

kubeadm 适合尽快得到集群；二进制适合要把每个进程参数摸清。可以先用 kubeadm 跑起来，再对照 `/etc/kubernetes/manifests` 里它写出的参数，在另一组机器上按二进制走一遍。
