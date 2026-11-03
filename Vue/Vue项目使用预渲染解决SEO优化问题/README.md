# 用预渲染补齐单页的首屏 HTML

单页打包后往往只剩一个空壳 `index.html`，爬虫不跑脚本就抓不到正文。构建期预渲染会在打包时把指定路由渲染成独立 HTML，再和脚本一起发布。常见组合是页面头部用 `vue-meta-info` 写标题与描述，构建链用 `prerender-spa-plugin` 调起无头浏览器访问路由。

只预渲染路由稳定的页面，例如关于、帮助、协议。文章详情这类路径随数据增长，不适合在构建时逐条生成。

```js
// vue.config.js
const PrerenderSPAPlugin = require('prerender-spa-plugin')
const path = require('path')

module.exports = {
  configureWebpack: {
    plugins: [
      new PrerenderSPAPlugin({
        staticDir: path.join(__dirname, 'dist'),
        routes: ['/', '/about', '/guide'],
      }),
    ],
  },
}
```

坑：插件要等页面里的异步数据落地再截 HTML。若接口在 `mounted` 才请求，截出来的仍是骨架。给目标路由加一个渲染完成标记，或只预渲染不依赖接口的静态页。另外每次改文案都要重新构建，否则搜索引擎看到的还是上一版快照。
