# 容器启动时改写前端接口地址

## 前言

镜像交付时，后端可以从配置中心读地址，换环境只改配置。前端一旦 `npm run build`，接口根路径就被写进打包结果，别人拉下镜像还得改源码、重装依赖、再打一遍包。小带宽机器上这套步骤很容易劝退。

做法改成：构建期不再把具体地址烤进业务代码，而是烤进「去读 `window._env_`」。容器启动时用脚本按环境变量重写这份配置，Nginx 再吐静态文件。同一份镜像可以指向不同的接口域名。

## 需要修改的文件

工程根目录新增或改动这些文件：

- `.env`：镜像内的默认地址
- `.default.env`：可被环境变量覆盖的兜底项，允许为空
- `env-config.js`：浏览器里先占一个空对象，避免脚本还没替换时直接报未定义
- `build/readEnv.js`：构建时把 `.env` 收成一段赋值语句
- `build/webpack.prod.conf.js`：定义 `process.env`，并把上述文件拷进 `dist`
- `index.html`：在头部插入那段赋值
- `env.sh`：容器入口，按真实环境变量重写配置
- `Dockerfile`：用 Nginx 镜像托管 `dist`，启动前先跑脚本

业务代码里原来写 `process.env.WEB_API` 的地方可以保留，构建后会变成读取全局对象上的同名字段。

## 添加.env文件

`.env` 只放地址，不放口令。下面是一套实验室环境的例子，换项目时整组替换，不要混进上一套系统的端口。

```bash
NODE_ENV=production
WEB_API=http://10.0.4.12:8603
FILE_API=http://10.0.4.12:8610/
MQ_CONSOLE=http://10.0.4.12:15672
FLOW_ADMIN=http://10.0.4.12:8070/
REGISTRY_API=http://10.0.4.12:8761
SEARCH_API=http://10.0.4.12:8605
ADMIN_API=http://10.0.4.12:8601
TRACE_UI=http://10.0.4.12:9411/
SQL_MONITOR=http://10.0.4.12:8601/druid/login.html
BOOT_BOARD=http://10.0.4.12:8606/wallboard
SITE_ORIGIN=http://10.0.4.12:9527
SEARCH_UI=http://10.0.4.12:5601
MEDIA_API=http://10.0.4.12:8602
```

这些键就是原先 `prod.env.js` 里导出的那一组。文件改走 `.env` 之后，`prod.env.js` 不再参与生产构建。

## 添加默认环境文件

再建一个 `.default.env`。它给「镜像里没写、运行时也可能不传」的键留位置。文件可以是空的，脚本仍会读它。空行要允许，解析时跳过。

```bash
# 可选兜底，留空表示沿用 .env
```

两个文件都不要提交真实公网地址到公开仓库时，改用内部示例网段，上线前在启动命令里覆盖。

## 添加 env-config.js文件

这个文件会被拷进站点根目录，也是脚本的输出目标。构建阶段先放一行占位：

```js
window._env_ = {};
```

占位让本地打开未替换的 `dist` 时至少不抛「读不到 `_env_`」。真正的键值由 `env.sh` 在容器里重写。

## 添加 getEnv.js文件

在 `build/readEnv.js` 里把 `.env` 收成一段可塞进 HTML 的脚本。只识别 `KEY=VALUE`，忽略空行和井号注释。值两端的引号剥掉。

```js
const fs = require('fs')
const path = require('path')

function loadPairs(filePath) {
  const text = fs.readFileSync(filePath, 'utf8')
  const pairs = {}
  text.split(/\r?\n/).forEach((line) => {
    const trimmed = line.trim()
    if (!trimmed || trimmed.charAt(0) === '#') return
    const eq = trimmed.indexOf('=')
    if (eq <= 0) return
    const key = trimmed.slice(0, eq).trim()
    let value = trimmed.slice(eq + 1).trim()
    if (
      (value.startsWith('"') && value.endsWith('"')) ||
      (value.startsWith("'") && value.endsWith("'"))
    ) {
      value = value.slice(1, -1)
    }
    pairs[key] = value
  })
  return pairs
}

function readEnv() {
  const filePath = path.resolve(process.cwd(), '.env')
  const pairs = loadPairs(filePath)
  return 'window._env_ = ' + JSON.stringify(pairs) + ';'
}

module.exports = readEnv
```

构建机上的 `.env` 决定 HTML 里的初始值。容器启动后 `env.sh` 会再覆盖一遍，所以镜像里这份初始值只是默认，不是最终地址。

## 更改webpack.prod.conf.js文件

### 核心代码

`DefinePlugin` 不要再展开 `prod.env.js`。把它替换成全局对象，这样业务包里的 `process.env.WEB_API` 会编译成 `window._env_.WEB_API`，运行时才取值。

`HtmlWebpackPlugin` 增加一个 `env` 选项，模板用它输出脚本。`CopyWebpackPlugin` 把 `.env`、脚本和占位文件带进 `dist`，否则容器里的入口脚本找不到它们。

```js
const readEnv = require('./readEnv')

new webpack.DefinePlugin({
  'process.env': 'window._env_'
})

new HtmlWebpackPlugin({
  filename: config.build.index,
  template: 'index.html',
  inject: true,
  favicon: resolve('favicon.ico'),
  title: '值班台',
  env: readEnv(),
  minify: {
    removeComments: true,
    collapseWhitespace: true,
    removeAttributeQuotes: true
  }
})

new CopyWebpackPlugin([
  {
    from: path.resolve(__dirname, '../static'),
    to: config.build.assetsSubDirectory,
    ignore: ['.*']
  },
  { from: path.resolve(__dirname, '../.env'), to: './' },
  { from: path.resolve(__dirname, '../env.sh'), to: './' },
  { from: path.resolve(__dirname, '../.default.env'), to: './' },
  { from: path.resolve(__dirname, '../env-config.js'), to: './' }
])
```

`process.env` 替换成不带引号的 `window._env_` 是整段方案的关键。若写成字符串 `'"window._env_"'`，业务代码会拿到一段文本，而不是对象。

### 完整代码

生产配置只保留和动态地址有关的插件，其余沿用模板原有的压缩和拆包。下面是合并后的插件与优化段，路径辅助函数和样式规则与原模板相同，这里不重复贴 loader。

```js
'use strict'
const path = require('path')
const webpack = require('webpack')
const config = require('../config')
const merge = require('webpack-merge')
const baseWebpackConfig = require('./webpack.base.conf')
const CopyWebpackPlugin = require('copy-webpack-plugin')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const readEnv = require('./readEnv')

function resolve(dir) {
  return path.join(__dirname, '..', dir)
}

const webpackConfig = merge(baseWebpackConfig, {
  mode: 'production',
  output: {
    path: config.build.assetsRoot,
    filename: 'js/[name].[chunkhash:8].js',
    chunkFilename: 'js/[name].[chunkhash:8].js'
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env': 'window._env_'
    }),
    new MiniCssExtractPlugin({
      filename: 'css/[name].[contenthash:8].css'
    }),
    new HtmlWebpackPlugin({
      filename: config.build.index,
      template: 'index.html',
      inject: true,
      title: '值班台',
      env: readEnv(),
      minify: { removeComments: true, collapseWhitespace: true }
    }),
    new CopyWebpackPlugin([
      { from: path.resolve(__dirname, '../static'), to: config.build.assetsSubDirectory, ignore: ['.*'] },
      { from: path.resolve(__dirname, '../.env'), to: './' },
      { from: path.resolve(__dirname, '../env.sh'), to: './' },
      { from: path.resolve(__dirname, '../.default.env'), to: './' },
      { from: path.resolve(__dirname, '../env-config.js'), to: './' }
    ])
  ]
})

module.exports = webpackConfig
```

若模板里还有 gzip 插件，保持原样接在这段后面。它和地址替换无关，不要为了动态配置把它删掉。

## 修改index.html

模板头部输出构建期生成的赋值。属性名用 `id="env"`，启动脚本靠这个 id 定位，再整段替换。

```html
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
  <title>值班台</title>
  <script id="env"><%= htmlWebpackPlugin.options.env %></script>
</head>
```

脚本必须出现在业务包之前。放在 `body` 底部、且晚于打包脚本时，入口代码可能先执行，读到的仍是占位空对象。

## 添加env.sh

容器启动时遍历 `.env` 和 `.default.env`。某个键若在进程环境里存在，用进程环境的值；否则用文件里的值。结果写成 `window._env_`，再替换 `index.html` 里 `id="env"` 的那一段。

```bash
#!/bin/bash
set -euo pipefail

root=$(cd "$(dirname "$0")" && pwd)
target="${root}/env-config.js"
: > "${target}"
echo "window._env_ = {" >> "${target}"

append_file() {
  local file="$1"
  [ -f "${file}" ] || return 0
  sed -i '/^[[:space:]]*$/d' "${file}"
  while IFS= read -r line || [ -n "${line}" ]; do
    case "${line}" in
      \#*) continue ;;
    esac
    key=${line%%=*}
    fallback=${line#*=}
    value=${!key-}
    if [ -z "${value}" ]; then
      value=${fallback}
    fi
    printf '  %s: "%s",\n' "${key}" "${value}" >> "${target}"
  done < "${file}"
}

append_file "${root}/.env"
append_file "${root}/.default.env"
echo "};" >> "${target}"

payload=$(tr -d '\n' < "${target}")
tmp="${root}/index.html.tmp"
sed "s#<script id=\"env\">.*</script>#<script id=\"env\">${payload}</script>#" \
  "${root}/index.html" > "${tmp}"
mv "${tmp}" "${root}/index.html"

exec "$@"
```

`set -u` 下读取可能不存在的环境变量要用 `${!key-}`，不能直接 `${!key}`，否则键未设置时脚本会退出，Nginx 起不来。替换用 `#` 当分隔符，是因为地址里含有 `/`，默认斜杠分隔会被截断。

Windows 上编辑过的脚本可能带 `\r`。镜像里执行前去掉回车，否则 shebang 找不到解释器。

## 编写Dockerfile

基础镜像用 Nginx，把 `dist` 拷到站点目录，入口改成先跑 `env.sh`，脚本末尾的 `exec` 再拉起 Nginx。

```dockerfile
FROM nginx:1.25-alpine
COPY ./dist/ /usr/share/nginx/html
RUN sed -i 's/\r$//' /usr/share/nginx/html/env.sh \
  && chmod +x /usr/share/nginx/html/env.sh
ENTRYPOINT ["/usr/share/nginx/html/env.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

Nginx 配置里的 `root` 要指向 `/usr/share/nginx/html`，并给前端路由加 `try_files` 回退。只换入口脚本、不改站点根目录时，页面会 404。

## 打包镜像

在前端工程根目录：

```bash
npm install
npm run build
docker build -t desk/admin-web:1.4 .
docker push desk/admin-web:1.4
```

推送前看一眼镜像里是否真有 `env.sh` 和 `.env`：`docker run --rm --entrypoint cat desk/admin-web:1.4 /usr/share/nginx/html/.env`。只有 `index.html` 没有这两个文件，说明 `CopyWebpackPlugin` 的路径没配对。

## 测试

### 默认配置

不传环境变量时，页面应请求 `.env` 里的地址。

```bash
docker run --name admin-web -d -p 9527:80 desk/admin-web:1.4
```

打开首页，在网络面板里看接口主机是否为 `10.0.4.12`。同时查看页面源代码，`id="env"` 的脚本应已经是对象字面量，而不是模板标记 `<%= ... %>`。还剩模板标记，说明 HtmlWebpackPlugin 没吃到 `env` 字段。

### 动态配置

启动时覆盖若干键。没出现在 `-e` 里的键继续用文件默认值。

```bash
docker run --name admin-web -d \
  -e WEB_API=http://10.8.0.20:8603 \
  -e MEDIA_API=http://10.8.0.20:8602 \
  -e SITE_ORIGIN=http://10.8.0.20:9527 \
  -p 9527:80 desk/admin-web:1.4
```

再打开首页，接口主机应变成 `10.8.0.20`。若只有 HTML 变了、请求仍打旧地址，多半是业务代码把地址写死成字符串，没有走 `process.env`。全文本搜索旧 IP，改成读环境字段后重新构建。容器已经创建过时，`docker rm -f admin-web` 再跑，否则名字冲突，你看到的仍是上一只容器。

还有几处和「地址没换」看起来很像、其实不是同一类问题：

- 浏览器用了旧的 `index.html`。给 `index.html` 单独加不缓存的响应头，哈希过的脚本可以长缓存。只缓存脚本、不缓存 HTML，启动脚本改写的那一段才会每次都是新的。
- `DefinePlugin` 写早了。开发环境的 webpack 配置若仍把 `WEB_API` 展开成字符串，本地是对的，生产镜像却不读 `window._env_`。两边配置要分开核对。
- 键名不一致。`.env` 里叫 `MEDIA_API`，组件里读 `process.env.VUE_APP_MEDIA`，运行时就是 `undefined`。以 `.env` 的键为唯一来源，组件不要再发明一套名字。
- 值里含有双引号或反斜杠。`env.sh` 用双引号包裹写进 JS，值本身再带引号会把对象截断，页面直接语法错误。地址类的值不要加引号；若必须包含特殊字符，改成 `JSON.stringify` 单个值再拼接。
- 健康检查打在入口脚本之前。编排系统若在 `env.sh` 还没把 HTML 替换完时就探测 80 端口，可能拿到半截文件。脚本是同步替换后再 `exec nginx`，一般来得及；若探测仍然过早，给探针加几秒启动等待。

本地不想打镜像时，可以先验证替换本身。准备一份假的 `index.html`，里面放 `<script id="env">window._env_ = {};</script>`，导出 `WEB_API` 后执行 `bash env.sh true`。`true` 代替 nginx，脚本替换完就退出。再打开 HTML，确认对象里已经是导出的地址。这一步通过之后，再进 Docker，能把脚本写错和镜像没拷到文件分开。

编排文件里覆盖地址时，键名要和 `.env` 左侧完全一致，包括大小写。写错一个字母不会报错，页面只是继续用镜像里的默认值，看起来像「动态配置没生效」。上线清单里把每个 `-e` 和 `.env` 的键对一遍，比在浏览器里猜 IP 更快。日志可以在 `env.sh` 末尾 `cat` 那份生成的脚本，但不要把带口令的值打出来。这套文件只适合放服务地址，不适合放数据库密码。

替换后的 HTML 若被 CDN 缓存，改环境变量重启容器也看不到新地址。站点入口页的缓存时间要短，或者发布时主动刷新入口。业务脚本文件名带哈希，可以缓存很久，它们只负责去读全局对象，地址不在脚本里。把入口和带哈希的脚本用同一条缓存规则罩住，是这套方案上线后最常见的改了却没生效。多副本同时启动时，每只容器各自改自己目录里的 HTML，不要把替换后的文件再写回镜像层。只读根文件系统会让替换写失败。入口脚本需要一个可写的站点目录，或者先把 HTML 拷到临时目录，再让 Nginx 的 root 指过去。临时目录在容器退出后消失，这是预期行为，不要把它挂成持久卷，否则下一只容器会读到上一套环境留下的地址，排查时会误以为脚本没有执行。
