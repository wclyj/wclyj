# Asciinema：用文本记录终端

## 前言

操作过程写成文档时，命令和输出容易对不齐。录成视频又不能复制屏幕上的命令，读者只能暂停后自己敲。Asciinema 把终端会话记成文本，播放时像在看操作过程，同时可以把屏幕上的命令选中复制。

## 介绍

项目主页：https://asciinema.org/

它的目标是用轻量的文本记录代替模糊的屏幕录像。在 Linux 上，它抓住终端的输入输出，保存后再按时间顺序回放。播放中可以暂停，并对画面里的文字做复制。Linux、macOS 和 BSD 都有客户端；Windows 需要放在 WSL 或别的 Unix 环境里用。

## 安装

### ubuntu 安装

Ubuntu 可以直接用发行版仓库，不必再加旧的 PPA：

```bash
sudo apt-get update
sudo apt-get install -y asciinema
```

发行版仓库里的版本偏旧时，再用官方安装脚本或 pip 安装当前版。

### CentOS 安装

```bash
sudo dnf install -y asciinema
```

CentOS 7 上如果没有这条包，改用 EPEL，或按官网的 pip 方式安装。

### Docker 安装

不想把客户端装进系统时，可以用容器。配置目录挂到本机，登录状态才不会随容器消失：

```bash
docker run --rm -it \
  -v "$HOME/.config/asciinema":/root/.config/asciinema \
  ghcr.io/asciinema/asciinema rec /tmp/session.cast
```

容器里录到的文件若要留在本机，把输出路径也挂出来。

## 快速开始

```bash
asciinema rec ~/casts/net-check.cast
# 在新开的 shell 里执行要展示的命令
exit
asciinema play ~/casts/net-check.cast
```

`rec` 会再开一层 shell，这层里的输入输出才被记录。`exit` 或 Ctrl+D 结束记录。播放时终端尺寸最好和录制时接近，否则换行位置会变。

## 上传到 Asciinema 官网

只存在本机的 cast 文件不方便发给别人。上传前先在 https://asciinema.org/ 用邮箱注册：页面会把登录链接发到邮箱，点开即完成，没有单独的密码框。以后登录同样是收一封邮件。

在终端绑定当前机器：

```bash
asciinema auth
# 用已经登录的浏览器打开打印出的链接
```

命令会打印一段一次性链接，用已经登录的浏览器打开，这台机器就和账号连上了。

绑定之后可以直接录并选择上传：

```bash
asciinema rec
# 操作结束后
exit
```

退出时按提示确认上传，终端会给出类似下面的地址：

```text
https://asciinema.org/a/demoLabCast91
```

打开该页即可在网页里播放。链接默认可能是公开的，含有口令或内网地址的录像不要上传。

## Asciinema 常见命令

```bash
# 打印绑定链接，浏览器里确认
asciinema auth

# 录制并在结束时询问是否上传
asciinema rec

# 只写到本地
asciinema rec ~/casts/pkg-install.cast

# 回放时把停顿压到最多 1.5 秒
asciinema play --idle-time-limit 1.5 ~/casts/pkg-install.cast

# 1.5 倍速回放
asciinema play --speed 1.5 ~/casts/pkg-install.cast

# 把整段输出打到标准输出，便于检索
asciinema cat ~/casts/pkg-install.cast

# 把已经存在的文件上传
asciinema upload ~/casts/pkg-install.cast
```

`--idle-time-limit` 适合命令之间想了很久的录像，避免观看时长时间停住。`--speed` 只影响播放，不改文件内容。

## 修改配置

默认配置路径：

```bash
${EDITOR:-vi} ~/.config/asciinema/config.toml
```

较新的客户端用 TOML。常用项如下，分号开头的行是说明：

```toml
[server]
# 自建服务时改这里；也可以用环境变量 ASCIINEMA_SERVER_URL
url = "https://cast.example.net"

[record]
command = "/bin/bash -l"
stdin = true
env = "SHELL,TERM,USER"
idle_time_limit = 1.5
yes = true
quiet = true

[play]
speed = 1.5
idle_time_limit = 1
```

只想限制停顿和播放速度时，留这两段即可：

```toml
[record]
idle_time_limit = 1.5

[play]
speed = 1.5
```

写好后重新 `rec`，新的录像会套用这些默认值，命令行上的同名参数仍然优先。

## 参考

- 项目文档：https://docs.asciinema.org/
- 主页：https://asciinema.org/
