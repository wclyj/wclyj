# 解决Vue项目打包后js文件过大的问题

## 前言

带宽只有 1M 时，首屏慢往往不是接口，而是包体。把 Vue、组件库拆到 CDN 之后，业务包里剩下的脚本仍可能有几百 KB。这一篇只处理「把构建产物再压成 gzip，并让 Nginx 直接吐出已经压好的文件」。

## 引入插件

使用 `compression-webpack-plugin` 在构建期生成 `.gz`。5.x 的选项名和 6.x 不一样，锁版本避免升级后配置失效。

```bash
npm i compression-webpack-plugin@5.0.1
```

在 `config/index.js` 打开开关，并限定只压脚本和样式：

```js
productionGzip: true,
productionGzipExtensions: ['js', 'css'],
```

`build/webpack.prod.conf.js` 里按开关挂插件。阈值设为 10KB，比这小的文件压完可能更大。`deleteOriginalAssets` 保持 false，否则不支持 gzip 的客户端没有回退文件。

```js
if (config.build.productionGzip) {
  const CompressionWebpackPlugin = require('compression-webpack-plugin')
  webpackConfig.plugins.push(
    new CompressionWebpackPlugin({
      filename: '[path].gz[query]',
      algorithm: 'gzip',
      test: new RegExp('\\.(' + config.build.productionGzipExtensions.join('|') + ')$'),
      threshold: 10240,
      minRatio: 0.8,
      deleteOriginalAssets: false
    })
  )
}
```

## 打包

```bash
npm run build
```

`dist` 里同一份脚本会多出一个同名 `.gz`。只有 `.gz`、浏览器仍在下原始文件，说明服务器没开静态预压缩。

## 修改Nginx配置

Vue 产物是静态文件，压缩与否由 Nginx 决定。`gzip_static on` 表示优先读已经存在的 `.gz`，而不是每个请求现压，CPU 不会被重复压缩吃掉。

```nginx
gzip on;
gzip_static on;
gzip_min_length 1k;
gzip_buffers 4 16k;
gzip_http_version 1.1;
gzip_comp_level 5;
gzip_types text/plain application/javascript text/css application/xml application/json;
gzip_vary on;
gzip_disable "MSIE [1-6]\\.";
```

改完执行 `nginx -s reload`。`gzip_static` 需要 Nginx 带 `ngx_http_gzip_static_module`，官方常见包已包含；若指令报未知，要换带该模块的构建。

## 测试

打开开发者工具的网络面板，选中业务脚本，看响应头是否带 `Content-Encoding: gzip`，再对比「传输大小」和「资源大小」。传输大小应明显小于资源大小。没有这个头，说明浏览器收到的仍是明文，前面的 `.gz` 文件没有被用上。缓存未失效时会看到旧的明文响应，先禁用缓存再刷新一次。
