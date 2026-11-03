# SPA 的 SEO 方案对比、最终实践

## 前言

Vue、React 做出来的站点大多是单页：体验顺，地址切换不整页刷新。搜索引擎拿到的第一份 HTML 却常常只有一个空节点。另外，数据要等脚本下完再请求，首屏也偏慢。

社区大致有四条路。下面按「浏览器里渲染、服务器上渲染、构建时生成、按访问者分流」比较，最后落到可插拔的动态渲染。

## 一、客户端渲染（CSR）方案

浏览器先收到几乎空白的 HTML，脚本执行完、接口返回后才把正文画出来。管理后台用这套很合适，因为没有收录需求。

**优点**

- 切换页面不用整页刷新，交互连续

**缺点**

- 不执行脚本的爬虫只能看到空节点，标题和正文都不在首包里
- 首屏要等包体和接口两段时间，弱网更明显

## 二、服务端渲染 （SSR）方案

在服务器跑一份和前端同构的渲染。请求进来时先把数据放进 store，产出带正文的 HTML，再交给浏览器把事件绑回去。爬虫即使不跑脚本也能读到标题和段落。

要补上两处单页里原本不存在的步骤：

1. Vuex 或 Redux 在浏览器里是进入页面后才有数据。服务端必须在回包前把同一份数据写入 store，否则 HTML 和注水后的状态对不上，页面会再闪一次。
2. 数据请求若写在 `created` 或 `componentDidMount`，服务端要能等待这些钩子里的异步结束，再序列化 HTML。

现成框架可以少写这些接线：Vue 常用 Nuxt，React 常用 Next。Angular 侧没有同等级的默认选项，一般自己接 Universal。不想上框架，就得改各自的 `render`，改动面比接入框架更大。

**优点**

- 首包里就有正文，对收录友好
- 热点页可以在服务端缓存 HTML，后面的请求直接回缓存

**缺点**

- 原有单页要按框架的目录和数据约定重排，迁一次成本高
- 部分只在浏览器存在的对象（`window`、局部缓存）要分成两端来写
- 每个访客都打到 Node 上时，渲染进程会变成新的瓶颈

## 三、构建时预渲染方案

打包阶段用无头浏览器把若干固定路由打开，把得到的 HTML 写进 `dist`。工具侧常见 `prerender-spa-plugin`，也可以直接用 Puppeteer 或已停更的 PhantomJS 自己写脚本。

它针对的是「整个站点只有一个 `index.html`」：构建结束后多出 `/about/index.html`、`/guide/index.html` 这种实体文件，首屏和这些固定页的收录会好转。

**优点**

- 业务代码几乎不用改，只在构建配置里列路由

**缺点**

- 路由数量跟着内容涨时没法在构建期穷举，例如每篇文章一个地址
- 正文一改就要重新构建，否则文件里还是旧文案

## 四、服务端动态渲染（利用user-agent）

单页照顾的是浏览器里的人，收录照顾的是爬虫。两边要的页面形态不一样，可以在入口按 `User-Agent` 分流：普通人拿单页壳子，爬虫拿已经跑完脚本的 HTML。

这不是给爬虫看一套、给人看另一套内容。Google 在动态渲染说明里写过：只要两边正文一致，就不把这种分流当成掩饰；若给人看「猫」、给爬虫看「狗」，才会被当成作弊。渲染失败时返回的错误页，也按普通错误页处理，不单独惩罚。

**优点**

- 人还是走单页，爬虫能读到正文

**缺点**

- 要多一个渲染服务。它只接待爬虫，流量远小于全站 SSR

几条路都试过之后，固定页用预渲染，正文页用动态渲染，不再把整站改成 SSR。手册站的章节地址稳定、正文一天改不了几次，这个组合比全站同构省事，也比纯单页能被收进去。

## 实现细节

浏览器请求先到 Nginx。普通 UA 回 `dist` 里的单页；爬虫 UA 反代到本机的渲染服务。渲染服务用无头浏览器打开原站，等正文节点出现，再把 HTML 交回去。静态资源仍回源，不进渲染。缓存放 Redis，避免每来一只爬虫都新开浏览器。

右路还可以再收：把渲染服务和静态站点合成一个 Node 进程，少一层反代。那是优化，不是第一版必须。

#### 社区方案：

Puppeteer 能驱动无头 Chrome，适合自己控制等待条件和样式内联。rendertron、prerender.io 把这件事包成服务，接入快，但等待条件和缓存策略不如自己写清楚。手册站选择自己用 Puppeteer 包一层，依赖控制在这几项：

```json
{
  "dependencies": {
    "express": "^4.18.2",
    "puppeteer": "^19.11.1",
    "ioredis": "^5.3.2"
  }
}
```

`server.js` 只做转发。渲染逻辑放在 `render.js`，方便单测时不拉起端口。

```js
import express from 'express'
import renderPage from './render.js'

const app = express()
const origin = 'https://docs.example.net'

app.get('/assets/*', (req, res) => {
  res.redirect(origin + req.originalUrl)
})

app.get('*', async (req, res, next) => {
  try {
    const { html, cost } = await renderPage(origin + req.originalUrl)
    res.set('Server-Timing', `render;dur=${cost}`)
    res.status(200).send(html)
  } catch (err) {
    next(err)
  }
})

app.listen(8790)
```

第一版缓存用内存 `Map`，方便对照问题出在渲染还是出在缓存：

```js
import puppeteer from 'puppeteer'

const memory = new Map()

export default async function renderPage(url) {
  if (memory.has(url)) return { html: memory.get(url), cost: 0 }
  const started = Date.now()
  const browser = await puppeteer.launch({
    args: ['--no-sandbox', '--disable-dev-shm-usage']
  })
  try {
    const page = await browser.newPage()
    await page.goto(url, { waitUntil: 'networkidle0', timeout: 20000 })
    await page.waitForSelector('#doc-body')
    const html = await page.content()
    memory.set(url, html)
    return { html, cost: Date.now() - started }
  } finally {
    await browser.close()
  }
}
```

这版能出 HTML，上线后会碰到四件事：

- 爬虫若会执行脚本，页面在浏览器里会再请求一次详情接口
- `Map` 没有过期，进程一重启全丢，跑得越久内存越大
- 样式仍是外链，不跑脚本的抓取快照是乱的
- `/assets/*.js` 也被当成页面去渲染，无头浏览器打开脚本内容，返回一堆乱码 HTML

### 重复请求

详情数据写在 `mounted` 里。无头浏览器执行了一次，HTML 回到一个同样会跑脚本的客户端时又执行一次。可以在正文容器上留一个标记，第二次发现标记已在就不再请求：

```js
mounted() {
  if (document.getElementById('doc-body')) return
  this.loadChapter().then((chapter) => {
    const holder = document.createElement('article')
    holder.id = 'doc-body'
    holder.textContent = chapter.title
    document.getElementById('app').appendChild(holder)
  })
}
```

只在「会执行脚本的爬虫」上有意义。完全不跑脚本的抓取不会进 `mounted`，多一次请求也不存在。标记必须是服务端 HTML 里已经有的节点，不能等客户端创建后才判断，否则第一次判断永远是空。

### 缓存机制

内存表换成 Redis，并给键一个过期时间。渲染进程重启后，下一次未命中才回源，不会把旧进程的内存无限留下。连接失败要打日志并退回现渲染，不要因为缓存挂了就让爬虫全部 500。

```js
import Redis from 'ioredis'

const redis = new Redis({
  host: '10.0.4.8',
  port: 6379,
  password: process.env.RENDER_REDIS_PASSWORD,
  maxRetriesPerRequest: 2
})

redis.on('error', (err) => {
  console.error('render cache', err.message)
})

export async function readCache(url) {
  try {
    return await redis.get('render:' + url)
  } catch (err) {
    return null
  }
}

export async function writeCache(url, html) {
  try {
    await redis.set('render:' + url, html, 'EX', 900)
  } catch (err) {
    console.error('render cache write', err.message)
  }
}
```

过期时间按正文更新频率定。手册一天改不了几次，15 分钟足够；若改完要立刻被收录，发布流程里对这个键做一次删除。

### 样式错乱

构建产物里的样式是这样挂的：

```html
<link rel="stylesheet" href="/assets/chapter.8f21aa.css">
```

不拉外链的抓取看不到这条规则，快照里的标题和正文挤在一起，排名侧也会把「页面不可用」算进去。渲染时把同源样式响应收进内存，再在页面里换成 `style` 标签。跨域字体不用内联，失败了也不要阻断正文。

```js
import puppeteer from 'puppeteer'
import { readCache, writeCache } from './cache.js'

export default async function renderPage(url) {
  const cached = await readCache(url)
  if (cached) return { html: cached, cost: 0 }
  const started = Date.now()
  const browser = await puppeteer.launch({
    args: ['--no-sandbox', '--disable-dev-shm-usage']
  })
  try {
    const page = await browser.newPage()
    const sheets = {}
    page.on('response', async (resp) => {
      const href = resp.url()
      const sameHost = new URL(href).host === new URL(url).host
      if (sameHost && resp.request().resourceType() === 'stylesheet') {
        sheets[href] = await resp.text()
      }
    })
    await page.goto(url, { waitUntil: 'networkidle0', timeout: 20000 })
    await page.waitForSelector('#doc-body')
    await page.$$eval('link[rel="stylesheet"]', (links, sheets) => {
      links.forEach((link) => {
        const css = sheets[link.href]
        if (!css) return
        const style = document.createElement('style')
        style.textContent = css
        link.replaceWith(style)
      })
    }, sheets)
    const html = await page.content()
    await writeCache(url, html)
    return { html, cost: Date.now() - started }
  } finally {
    await browser.close()
  }
}
```

内联之后 HTML 会变大。缓存值要看 Redis 的单值上限，超长页可以只内联首屏用到的那一个包，其余链接保留。

### 错误渲染

无头页面里的懒加载可能再去请求 `/assets/chapter.8f21aa.js`。若渲染服务的兜底路由把所有路径都打开浏览器，这个脚本地址会被当成 HTML 页面再渲染一次。静态前缀直接回源：

```js
import express from 'express'
import request from 'request'
import renderPage from './render.js'

const app = express()
const origin = 'https://docs.example.net'
const pass = ['/assets/*', '/favicon.ico', '/manifest.json', '/logo*']

pass.forEach((rule) => {
  app.get(rule, (req, res) => {
    request(origin + req.originalUrl).pipe(res)
  })
})

app.get('*', async (req, res) => {
  const { html, cost } = await renderPage(origin + req.originalUrl)
  res.set('Server-Timing', `render;dur=${cost}`)
  res.status(200).send(html)
})

app.listen(8790)
```

和整站 SSR 比，这层服务只在爬虫进来时工作，单页代码除了「已有正文标记就别再请求」之外不用改。标记也可以不做，重复请求通常只是多打一次读接口。

## Nginx 配置

分流放在站点配置里，命中爬虫关键字才反代到渲染端口，其余走静态根目录。关键字过宽会把普通浏览器送去渲染，端口会被打满。

```nginx
map $http_user_agent $to_render {
  default 0;
  ~*(googlebot|bingbot|baiduspider|360Spider|Sogou) 1;
}

server {
  listen 80;
  root /var/www/docs;
  location / {
    if ($to_render = 1) {
      proxy_pass http://127.0.0.1:8790;
    }
    try_files $uri $uri/ /index.html;
  }
}
```

`if` 配在 `location` 里要小心和 `try_files` 的执行顺序。更稳的写法是拆两个 `location`，用 `error_page` 或单独的内部跳转，避免 `if` 把静态文件逻辑截断。改完用带 UA 和不带 UA 的两条 curl 对照。

## 附录

#### 常见爬虫 user-agent

| 主体 | user-agent | 用途 |
| --- | --- | --- |
| Google | googlebot | 网页检索 |
| Google | google-structured-data-testing-tool | 结构化数据检测 |
| Google | Mediapartners-Google | 广告匹配时的抓取 |
| Microsoft | bingbot | 网页检索 |
| LinkedIn | linkedinbot | 分享卡片预览 |
| 百度 | baiduspider | 网页检索 |
| 奇虎 360 | 360Spider | 网页检索 |
| 搜狗 | Sogou Spider | 网页检索 |
| Yahoo | Yahoo! Slurp China | 中文检索 |
| Yahoo | Yahoo! Slurp | 网页检索 |
| Twitter | twitterbot | 卡片预览 |
| Facebook | facebookexternalhit | 链接预览 |
| — | rogerbot | 站点审计类抓取 |
| — | embedly | 链接摘要 |
| Quora | quora link preview | 问答站预览 |
| — | showyoubot | 内容聚合 |
| — | outbrain | 推荐组件抓取 |
| — | pinterest | 图钉预览 |
| — | slackbot | 会话里的链接展开 |
| — | vkShare | 社交分享预览 |
| — | W3C_Validator | 标记校验 |

预览类机器人（聊天工具、社交分享）也会来抓标题和首图。若只把搜索引擎送去渲染，分享卡片上的摘要会是空的。可以把 `twitterbot`、`slackbot` 一并放进 `map`。

#### 模拟爬虫测试

```bash
curl -s https://docs.example.net/guide/install | head
curl -s -A 'Googlebot' https://docs.example.net/guide/install | head
```

第一条的 HTML 里应几乎没有章节正文；第二条应含有章节标题。两条都空，说明分流没生效或渲染服务没等到 `#doc-body`。第二条返回脚本源码，说明静态前缀被送进了渲染。渲染端口只应出现在反代后面，不要单独暴露到公网，否则任何人都能拿它把任意地址跑成 HTML，浏览器进程会被打满。缓存键要把查询参数算进去，同一路径不同参数可能是两篇文章。只按路径缓存时，后一次抓取会拿到前一篇的正文。

#### 参考资料

1. 构建期预渲染与首屏：美团技术文章里关于首帧的实践记录
2. Google 搜索文档中的 dynamic rendering 说明
3. Google 抓取用到的用户代理列表。列表会更新，上线前对照官方页核对关键字，避免漏掉新的抓取标识。
