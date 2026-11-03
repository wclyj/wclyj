# vue打包后的dist文件如何启动测试

## 前言

`npm run build` 之后得到 `dist`。双击里面的 `index.html` 会白屏，因为资源路径和前端路由都按站点根目录来写，文件协议打开时对不上。本地验收可以用 Nginx 做静态映射，也可以用一个专门的静态服务器。

## nginx静态代理

在 `nginx.conf` 里加一段，把站点根指到构建目录，并放行预检请求，避免本地调接口时被浏览器拦住。

```nginx
server {
  listen 8600;
  server_name localhost;
  add_header Access-Control-Allow-Origin *;
  add_header Access-Control-Allow-Methods *;
  add_header Access-Control-Allow-Headers *;
  if ($request_method = 'OPTIONS') {
    return 204;
  }
  location / {
    root /data/desk/dist;
    index index.html;
    try_files $uri $uri/ /index.html;
  }
}
```

把本次构建的 `dist` 内容放到 `root` 指向的目录，重载 Nginx 后访问：

```text
http://localhost:8600
```

`try_files` 回退到 `index.html`，刷新子路由才不会 404。少了这一行，首页能开，点进内页再刷新就掉到 Nginx 的错误页。

## server方式启动

不想起 Nginx 时，用 `serve` 把目录当成单页站点托管。`-s` 表示找不到文件时回退到 `index.html`。

```bash
npm install -g serve
serve -s dist -l 8600
```

然后打开 `http://localhost:8600`。新版本可以用 `-l` 指定端口，不必固定在 5000。若公司环境不允许全局安装，在项目里执行 `npx serve -s dist -l 8600`，效果相同。
