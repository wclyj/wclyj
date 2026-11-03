# Docker Desktop 入门简介

## 参考

安装条件以产品文档为准：https://docs.docker.com/desktop/

## 介绍

Docker Desktop 给 Windows 和 macOS 提供图形化容器环境。Windows 上新安装包走 WSL2，专业版也可改用 Hyper-V。固件里要先打开硬件虚拟化。

## 下载

到 Docker 网站按系统下载安装包。Windows 选 exe，Mac 要分清苹果芯片和 Intel 芯片。

## 安装

打开“启用或关闭 Windows 功能”，勾选“适用于 Linux 的 Windows 子系统”和“虚拟机平台”。改用 Hyper-V 时再勾选 Hyper-V，按提示重启。然后运行安装包。向导会检查虚拟化、系统版本和 WSL 是否齐全。

## 运行

从开始菜单打开 Docker Desktop。若提示 WSL 内核过旧，先装微软提供的内核更新包，再重新打开。图标变为运行中后，在 PowerShell 执行：

```powershell
docker version
```

客户端和引擎的版本都能打印，就可以把 Linux 上用过的 pull、run、compose 拿到这里做。内存和 CPU 配额在设置页调整。
