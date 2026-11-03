# 用 Nginx 配 HTTPS

## 前言

小程序和不少浏览器接口要求整站走 TLS。应用自己不一定要处理证书，可以在前面加一层 Nginx：对外 443，对内仍转发到本机的 HTTP 端口。

## 证书购买

证书向云厂商申请。自签证书能把信道加密，但浏览器不信任签发者，访问时仍会提示风险，不适合对外部用户。

免费的单域名证书在腾讯云、阿里云的证书服务里都能申请，额度以控制台当年的说明为准。免费证书通常只覆盖一个完全限定域名。`api.example.net` 和 `img.example.net` 要各申请一张，不能指望一张证书顺带盖住第二个名字，除非买的是通配符。

申请时填这次要保护的域名，例如图片域名 `img.example.net`。验证方式选 DNS：控制台会给出一条主机记录和一段校验值。

到域名所在的解析服务（域名买在哪家就去哪家）新增一条记录，类型和取值按控制台复制，主机记录一般是 `_dnsauth` 一类前缀。保存后等解析生效，再回证书页面点校验。

校验通过之前，还要有一条普通的 A 记录，把 `img.example.net` 指到 Nginx 所在主机的公网地址。只有校验记录、没有 A 记录时，证书能签发，用户却访问不到服务器。

签发完成后下载 Nginx 格式的压缩包。包里是证书链和私钥。在服务器上建目录并放进去，权限收紧：

```bash
mkdir -p /etc/edge/certs/img
install -m 644 img.example.net.crt /etc/edge/certs/img/fullchain.pem
install -m 600 img.example.net.key /etc/edge/certs/img/privkey.pem
```

Nginx 若是自己编译的，需要带 SSL 模块。编译参数里加上 `--with-http_ssl_module`，安装前缀和临时目录按机器实际情况填写，例如：

```bash
./configure \
  --prefix=/opt/edge \
  --with-http_ssl_module \
  --with-http_gzip_static_module \
  --error-log-path=/var/log/edge/error.log \
  --http-log-path=/var/log/edge/access.log \
  --http-client-body-temp-path=/var/cache/edge/client \
  --http-proxy-temp-path=/var/cache/edge/proxy
```

然后 `make && make install`。用系统包安装的 Nginx 通常已经带 SSL，不必重编，直接改配置。

`/opt/edge/conf/nginx.conf` 里增加：

```nginx
server {
    listen 443 ssl;
    server_name img.example.net;

    ssl_certificate     /etc/edge/certs/img/fullchain.pem;
    ssl_certificate_key /etc/edge/certs/img/privkey.pem;
    ssl_session_timeout 15m;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://127.0.0.1:8612;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

`nginx -t` 通过后执行 `nginx -s reload`。用浏览器打开 `https://img.example.net`，证书链应被信任，请求被转到本机 8612。

第二个域名，例如 `api.example.net`，重复申请、解析和摆放证书，再加一段 server，不要和上一张证书混用文件：

```nginx
server {
    listen 443 ssl;
    server_name api.example.net;

    ssl_certificate     /etc/edge/certs/api/fullchain.pem;
    ssl_certificate_key /etc/edge/certs/api/privkey.pem;
    ssl_session_timeout 15m;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8616;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

两段 `server` 都监听 443，靠 `server_name` 区分。证书路径写错时，`nginx -t` 会直接指出打不开的文件。
