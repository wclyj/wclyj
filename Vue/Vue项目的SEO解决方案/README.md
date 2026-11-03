# SPA 的 SEO 方案对比、最终实践

> 单页把路由和数据都放进浏览器之后，首包往往只剩一个挂载点。检索程序若不会跑脚本，就收不到商品名和描述；人在弱网上还要先等包、再等接口。下面按四类做法比较，并记录商品目录站最后采用的按访问者分流。

## 一、客户端渲染（CSR）方案

目录页的 HTML 在离开源站时没有商品名。名称来自接口，接口要等包执行后才发出。后台系统可以接受这种空白首包，面向搜索的目录页不行。

**优点**

- 筛选、翻页都在当前页完成，不用整站刷新

**缺点**

- 不会执行脚本的抓取看到的是空挂载点，商品名进不了索引
- 首屏时间等于包体时间加上接口时间

## 二、服务端渲染 （SSR）方案

在 Node 里把同一套页面先画出来：请求到达后先取商品 JSON，写进 store，再输出 HTML。抓取读到的是已经填好的标题和价格。

落地时有两件单页不必考虑的事：

1. 状态库在浏览器启动前是空的。服务端要先灌入这件商品的数据，再把同一份状态随 HTML 交出去，否则客户端注水时会把已有 DOM 再刷掉。
2. 取数若放在 `created` 或 `componentDidMount`，渲染过程必须等这些异步结束。不等的话，输出的仍是骨架。

Vue 工程可迁到 Nuxt，React 工程可迁到 Next。自己改 `render` 也能做，但路由、状态和数据预取都要重接一遍。

**优点**

- 首包含正文，收录和首屏都受益
- 热点商品的 HTML 可以缓存，后续请求不再跑组件

**缺点**

- 目录站原来的页面结构要按框架重排，迁移面大
- 只在浏览器里才有的对象要拆到客户端分支
- 所有访客都经过渲染进程时，机器成本接近再做一套后端

## 三、构建时预渲染方案

打包时用无头浏览器打开「关于」「配送说明」这类固定地址，把结果写成独立 HTML。`prerender-spa-plugin` 或 Puppeteer 脚本都能做。

它解决的是全站只有一份 `index.html` 的问题，适合页数少、文案稳定的路由。

**优点**

- 页面组件基本不动，只在构建配置里登记路径

**缺点**

- 商品详情的路径随 SKU 增长，构建期无法逐条生成
- 价格或文案变更后，必须重新打包，文件才跟得上

## 四、服务端动态渲染（利用user-agent）

普通访客继续要单页的筛选体验，检索程序要的是带商品名的 HTML。入口可以看 `User-Agent`：浏览器拿 `dist`，检索程序拿渲染后的结果。

有人担心两套响应会被当成作弊。检索方的说明可以概括成：两边正文一致就不算掩饰；渲染过程中偶尔吐出的错误页按一般错误处理；若给人看商品、给抓取看另一类完全不同的页面，才会被认定成掩饰。

**优点**

- 人走单页，抓取能读到商品名和描述

**缺点**

- 需要一个只服务抓取流量的渲染进程，量级远小于全站 SSR

目录站的固定说明页用构建期预渲染，商品详情走动态渲染，整站不迁到 SSR。价格会变，所以详情页的缓存要短；配送说明几乎不变，留在构建产物里即可，不必每条 SKU 都进无头浏览器。

## 实现细节

Nginx 放在最外层。未命中检索关键字时直接读静态目录；命中时把请求交给本机渲染端口。渲染端口打开的仍是对外站点，这样接口、登录态和静态域名与线上一致。缓存用 Redis，键是完整地址。

架构上还能把静态托管并进 Node，少维护一份 Nginx。第一版保持反代，出问题好单独重启渲染进程。

#### 社区方案：

自建选用 Puppeteer。托管型的 rendertron、prerender.io 适合先验证效果，等待选择器和缓存时间不容易按商品页调整。目录站的依赖：

```json
{
  "dependencies": {
    "express": "^4.18.2",
    "puppeteer": "^21.6.1",
    "redis": "^4.6.11"
  }
}
```

入口只负责路由，打开浏览器的细节放在单独模块：

```js
import express from 'express'
import paint from './paint.js'

const app = express()
const site = 'https://shop.example.net'

app.get('*', async (req, res) => {
  const painted = await paint(site + req.path)
  res.set('Server-Timing', `paint;dur=${painted.ms}`)
  res.type('html').send(painted.markup)
})

app.listen(8811)
```

基础渲染先用进程内 Map，确认选择器能等到商品标题：

```js
import puppeteer from 'puppeteer'

const local = new Map()

export default async function paint(url) {
  if (local.has(url)) return { markup: local.get(url), ms: 0 }
  const t0 = Date.now()
  const browser = await puppeteer.launch({ headless: 'new', args: ['--no-sandbox'] })
  const page = await browser.newPage()
  await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 15000 })
  await page.waitForSelector('[data-sku-title]')
  const markup = await page.content()
  await browser.close()
  local.set(url, markup)
  return { markup, ms: Date.now() - t0 }
}
```

试跑时记下四个缺口：

- 会执行脚本的抓取把 HTML 拿回去后，详情接口又打了一次
- Map 无过期，重启即丢，商品一多内存跟着涨
- 快照里样式丢失，因为 CSS 还是外链
- 带哈希的脚本地址被兜底路由当成商品页打开

### 重复请求

商品数据在 `mounted` 里拉取。无头浏览器已经拉过，回到仍会跑脚本的环境会再拉一次。详情节点带上 `data-sku-title` 后，客户端见到它就跳过请求：

```js
async mounted() {
  if (this.$el.querySelector('[data-sku-title]')) return
  const item = await this.fetchItem(this.$route.params.sku)
  this.item = item
}
```

选择器要和渲染服务等待的是同一个。服务等的是 A、客户端判断的是 B，会出现「渲染等到了，客户端仍重新请求」或反过来一直空白。

### 缓存机制

Map 换成 Redis，写入时带上秒级过期。价格变动频繁的类目把过期调短；说明页可以更长。连接被拒绝时记录日志并继续渲染，不把缓存故障放大成全站抓取失败。

```js
import { createClient } from 'redis'

const client = createClient({ url: process.env.RENDER_REDIS_URL })
client.on('error', (err) => console.error('sku-render', err.message))
await client.connect()

export async function take(url) {
  return client.get('sku:' + url)
}

export async function save(url, markup) {
  await client.set('sku:' + url, markup, { EX: 480 })
}
```

重试不要无限循环。地址写错时快速失败，比卡住渲染端口更安全。密钥放环境变量，不要写进仓库。

### 样式错乱

商品页的样式链接类似：

```html
<link rel="stylesheet" href="/static/sku.91ab0c.css">
```

不拉 CSS 的快照会把价格和参数挤成一行。渲染过程中收集同源样式文本，在序列化前替换成内联 `style`。第三方统计脚本不要内联，也不要等待它结束。

```js
import puppeteer from 'puppeteer'
import { take, save } from './store.js'

export default async function paint(url) {
  const hit = await take(url)
  if (hit) return { markup: hit, ms: 0 }
  const t0 = Date.now()
  const browser = await puppeteer.launch({ headless: 'new', args: ['--no-sandbox'] })
  try {
    const page = await browser.newPage()
    const bucket = {}
    page.on('response', async (resp) => {
      if (resp.request().resourceType() !== 'stylesheet') return
      if (new URL(resp.url()).host !== new URL(url).host) return
      bucket[resp.url()] = await resp.text()
    })
    await page.goto(url, { waitUntil: 'networkidle2', timeout: 15000 })
    await page.waitForSelector('[data-sku-title]')
    await page.$$eval('link[rel="stylesheet"]', (nodes, bucket) => {
      nodes.forEach((node) => {
        const text = bucket[node.href]
        if (!text) return
        const style = document.createElement('style')
        style.appendChild(document.createTextNode(text))
        node.replaceWith(style)
      })
    }, bucket)
    const markup = await page.content()
    await save(url, markup)
    return { markup, ms: Date.now() - t0 }
  } finally {
    await browser.close()
  }
}
```

内联使单页 HTML 变大。Redis 内存要按「详情页数量 × 单页体积 × 过期窗口」估算，不够就把过期缩短，而不是关掉内联。

### 错误渲染

懒加载会再请求 `/static/sku.91ab0c.js`。兜底路由若一视同仁，这个地址会被打开成「页面」。静态前缀和站点图标直接回源：

```js
import express from 'express'
import request from 'request'
import paint from './paint.js'

const app = express()
const site = 'https://shop.example.net'

function pipe(req, res) {
  request(site + req.originalUrl).pipe(res)
}

app.get('/static/*', pipe)
app.get('/favicon.ico', pipe)
app.get('/manifest.json', pipe)
app.get('/brand*', pipe)

app.get('*', async (req, res) => {
  const painted = await paint(site + req.originalUrl)
  res.set('Server-Timing', `paint;dur=${painted.ms}`)
  res.status(200).send(painted.markup)
})

app.listen(8811)
```

相对整站 SSR，这层只在抓取到来时工作，商品页组件除了跳过已渲染的二次取数，不必改写。二次取数本身通常无害，跳过是为了省接口额度。

## Nginx 配置

按用户代理分流的写法可以参考常见的 prerender 片段：用 `map` 把检索关键字打成变量，命中时 `proxy_pass` 到渲染端口，未命中时 `try_files` 回单页。关键字列表与附录一致即可，不要把空 UA 也送去渲染，否则本地 curl 和健康检查都会打到无头浏览器。

## 附录

#### 常见爬虫 user-agent

| 主体 | user-agent | 用途 |
| --- | --- | --- |
| Google | googlebot | 检索收录 |
| Google | google-structured-data-testing-tool | 校验结构化数据 |
| Google | Mediapartners-Google | 广告素材相关抓取 |
| Microsoft | bingbot | 检索收录 |
| LinkedIn | linkedinbot | 生成分享摘要 |
| 百度 | baiduspider | 检索收录 |
| 奇虎 360 | 360Spider | 检索收录 |
| 搜狗 | Sogou Spider | 检索收录 |
| Yahoo | Yahoo! Slurp China | 中文结果抓取 |
| Yahoo | Yahoo! Slurp | 检索收录 |
| Twitter | twitterbot | 生成卡片 |
| Facebook | facebookexternalhit | 生成预览 |
| — | rogerbot | 审计类访问 |
| — | embedly | 抽取摘要 |
| Quora | quora link preview | 问答引用预览 |
| — | showyoubot | 聚合抓取 |
| — | outbrain | 推荐位抓取 |
| — | pinterest | 图片书签预览 |
| — | slackbot | 展开链接 |
| — | vkShare | 分享抓取 |
| — | W3C_Validator | 校验文档 |

商品分享若依赖聊天工具里的标题和主图，要把预览类代理也送进渲染，只放搜索引擎关键字时，聊天窗口里仍是空卡片。

#### 模拟爬虫测试

```bash
curl -s https://shop.example.net/item/sku-18 | head
curl -s -A 'Baiduspider' https://shop.example.net/item/sku-18 | head
```

不带代理的响应里不应出现商品名。带上检索代理后，响应里应有 `data-sku-title` 对应的标题文本。两条都含标题，说明静态文件本身已经是预渲染结果，分流没有被测到。第二条很慢且没有标题，说明选择器一直没出现，先把超时和接口地址查清。渲染服务的浏览器进程要限制并发，商品页一多时同时开几十个页面会把内存吃光。用一个长度为 2 的队列即可，检索本身不是实时流量。商品下架后要删掉对应缓存键，否则抓取在过期前仍会看到已下架的标题和价格。删除可以放在商品保存接口的收尾，按详情地址拼出和渲染服务相同的键。键的前缀两边要约定好，渲染服务用 `sku:` 而保存接口删 `render:`，过期前下架商品仍会被抓取到。把前缀写成同一个常量，放在两边都能读到的配置里。常量不一致时，删除看起来成功，抓取仍命中旧值，直到过期才消失。联调时先读一次这个键，确认删的和写的是同一条。

#### 参考资料

1. 首屏与构建期预渲染的公开实践记录（美团技术团队的首帧文章）
2. Google 搜索中心对 dynamic rendering 的说明
3. Google 公布的抓取用户代理一览
