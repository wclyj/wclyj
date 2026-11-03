# CentOS 下从源码安装 Nginx

## 1、下载 nginx

发布页：http://nginx.org/download/

选一个稳定版源码包，例如 `nginx-1.26.2.tar.gz`，放到 `/usr/local/src`。不要用来历不明的二进制包代替官方源码。

## 2、使用解压命令进行解压

```bash
cd /usr/local/src
tar -xzf nginx-1.26.2.tar.gz
cd nginx-1.26.2
```

解压后目录里应有 `configure` 和 `src`。

## 3、准备编译要用的依赖

### 装上编译器

```bash
dnf install -y gcc gcc-c++ make
```

没有编译器时，`configure` 还能跑完，`make` 会在第一份源文件上报错。

### 安装第三方开发包

PCRE 提供和 Perl 兼容的正则库。Nginx 的 `location` 匹配依赖它，所以要装库和头文件。`pcre-devel`（或 `pcre2-devel`）才带编译所需的头文件，只装运行库不够。

```bash
dnf install -y pcre2 pcre2-devel
```

zlib 提供压缩算法。开启 gzip 时，响应体会先压缩再发出，因此需要 zlib 的头文件。

```bash
dnf install -y zlib zlib-devel
```

OpenSSL 提供 TLS。只做 HTTP 可以不装，要听 443 就必须在编译时打开 SSL 模块，并装上开发包。

```bash
dnf install -y openssl openssl-devel
```

依赖装完后仍留在源码目录，生成 Makefile：

```bash
./configure \
  --prefix=/opt/edge \
  --pid-path=/run/edge/nginx.pid \
  --lock-path=/run/edge/nginx.lock \
  --error-log-path=/var/log/edge/error.log \
  --http-log-path=/var/log/edge/access.log \
  --with-http_gzip_static_module \
  --http-client-body-temp-path=/var/cache/edge/client \
  --http-proxy-temp-path=/var/cache/edge/proxy \
  --http-fastcgi-temp-path=/var/cache/edge/fastcgi \
  --http-uwsgi-temp-path=/var/cache/edge/uwsgi \
  --http-scgi-temp-path=/var/cache/edge/scgi
```

需要 HTTPS 时，在同一条 `configure` 末尾加上 `--with-http_ssl_module`，其余路径保持一致，然后重新执行。漏了这个开关时，配置文件里的 `ssl_certificate` 会被拒绝，只能重新配置并编译。

对应的站点可以写成：

```nginx
server {
    listen 443 ssl;
    server_name console.example.net;
    ssl_certificate     /etc/edge/certs/console.example.net.crt;
    ssl_certificate_key /etc/edge/certs/console.example.net.key;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;
    root /srv/console/dist;
}
```

`ssl on;` 在近期版本里已经废弃，用 `listen 443 ssl;` 即可。

`configure` 把临时目录指到了 `/var/cache/edge`。这些目录不会自动创建，启动前要建好，否则报错信息会指向 client body 的临时路径：

```bash
mkdir -p /var/cache/edge/client /run/edge /var/log/edge
```

然后编译并安装：

```bash
make -j"$(nproc)"
make install
```

安装结果在 `/opt/edge`，可执行文件是 `/opt/edge/sbin/nginx`。

## 启动 Nginx

```bash
/opt/edge/sbin/nginx
ss -lntp | grep ':80'
```

`ss` 能看到 master 进程监听 80，即启动成功。配置改完后不要先停再起，用 reload 让旧进程处理完手上的连接。

## nginx 常用命令

```bash
# 启动
/opt/edge/sbin/nginx

# 立刻停止，在途请求会被断开
/opt/edge/sbin/nginx -s stop

# 处理完已接受的请求再退出
/opt/edge/sbin/nginx -s quit

# 重新加载配置，进程号保持
/opt/edge/sbin/nginx -s reload

# 改配置后先做语法检查
/opt/edge/sbin/nginx -t
```

`-t` 通过后再 reload。失败时看 `/var/log/edge/error.log` 的最后几行。
