# 自签证书，并让服务端校验证书

## 前言

线上通信不要裸奔。给 gRPC 套上 TLS 时，开发机可以自己签发一对密钥和证书，不必先买商业证书。

Linux 与 macOS 多数已经带有 `openssl`。Windows 需要单独安装 Win32 OpenSSL，安装后确认 `bin` 已进入 PATH，在终端执行 `openssl version` 能打出版本号即可。

## 生成步骤

下面四条在项目的 `certs` 目录执行。口令只用于演示，不要沿用到生产。

```bash
openssl genrsa -aes256 -out svc.key 2048
openssl req -new -key svc.key -out svc.csr -subj "/CN=api.lab.local"
openssl rsa -in svc.key -out svc.nopass.key
openssl x509 -req -days 365 -in svc.csr -signkey svc.nopass.key -out svc.crt
```

第一条生成带口令的私钥；第二条做证书请求，通用名写成服务域名；第三条导出一份去掉口令的私钥，方便进程启动时不必交互输入；第四条用该私钥自签出证书。

服务端加载 `svc.crt` 与 `svc.nopass.key` 构造 TLS 凭证；客户端要么信任这张自签证书，要么跳过校验只用于本机联调。到这里，自签材料就齐了。
