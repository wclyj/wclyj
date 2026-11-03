# Vue项目如何关闭Eslint校验

## 前言

dev 一启动就铺满规则报错，保存也被格式卡住。规则过严、又暂时没人收口时，可以先把构建期的校验摘掉，让页面能跑起来。这不是删掉代码规范，只是构建不再因为风格问题中断。

## 关闭Eslint

Vue CLI 2 的工程里，规则挂在 `build/webpack.base.conf.js` 的 `createLintingRule`。把 loader 相关字段留空，预加载阶段就不再跑 `eslint-loader`。

```js
const createLintingRule = () => ({
  // test: /\.(js|vue)$/,
  // loader: 'eslint-loader',
  // enforce: 'pre',
  // include: [resolve('src'), resolve('test')],
  // options: {
  //   formatter: require('eslint-friendly-formatter'),
  //   emitWarning: !config.dev.showEslintErrorsInOverlay
  // }
})
```

改完重新执行 `npm run dev`，覆盖层里的风格报错会消失。

Vue CLI 3 以后没有这份 webpack 文件，改 `vue.config.js`：

```js
module.exports = {
  lintOnSave: false
}
```

不要只删 `package.json` 里的 `lint` 脚本，开发服务器仍会在保存时调用同一套规则。编辑器插件若单独启用了 ESLint，IDE 里还会画红线，那是编辑器配置，和这次构建开关无关。
