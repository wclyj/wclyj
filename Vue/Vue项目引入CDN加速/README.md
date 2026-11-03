# Vue项目引入CDN加速

## 前言

小带宽机器上，把 Vue、组件库和富文本打进业务包，首屏要等这些大文件从源站慢慢下来。把它们改成公共 CDN 上的脚本，业务包只保留自己的代码，源站压力会小一截。版本号要写死，不要用「最新」地址，否则某一天上游升级会让页面直接白屏。

## 引入Vue和Element

Element 文档提供了样式和脚本的 CDN 地址。在 `index.html` 里按样式、Vue、组件库的顺序引入。Vue 必须在 Element 之前，因为组件库会读取全局的 `Vue`。

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
  <title>值班台</title>
  <link rel="stylesheet" href="https://unpkg.com/element-ui@2.15.14/lib/theme-chalk/index.css">
</head>
<body>
  <div id="app"></div>
  <script src="https://unpkg.com/vue@2.6.14/dist/vue.min.js"></script>
  <script src="https://unpkg.com/element-ui@2.15.14/lib/index.js"></script>
</body>
</html>
```

版本号和 `package.json` 里锁定的保持一致，否则本地构建和线上脚本不是同一份 API。

## 引入Ckeditor

富文本同样外置。到 BootCDN 或 cdnjs 上找与锁文件相同的版本，例如：

```html
<script src="https://cdn.bootcdn.net/ajax/libs/ckeditor/4.16.2/ckeditor.js"></script>
```

BootCDN 由 Bootstrap 中文网维护，收录内容主要同步 cdnjs。生产环境若对稳定性要求高，把文件下到自己的对象存储，CDN 只作开发期加速。

## 修改main.js文件

全局对象已经由脚本创建，入口不要再 `import` 一遍，否则打包结果里会有第二份 Vue。

```js
// 运行时由 index.html 的脚本提供
// import ElementUI from 'element-ui'
// import 'element-ui/lib/theme-chalk/index.css'
// Vue.use(ElementUI)
```

## 修改webpack.base.conf.js

告诉 webpack：这些包不要打进 bundle，运行时去全局变量上取。键是 import 时的包名，值是全局变量名。Element 的全局名是 `ELEMENT`，Vue 是 `Vue`，CKEditor 是 `CKEDITOR`。

```js
externals: {
  vue: 'Vue',
  'element-ui': 'ELEMENT',
  CKEDITOR: 'window.CKEDITOR'
}
```

漏配 `externals` 时，页面会同时加载 CDN 脚本和打包进去的那一份，体积没有下降，还可能出现两个 Vue 实例导致组件注册失效。
