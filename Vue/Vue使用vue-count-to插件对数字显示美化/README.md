# Vue使用vue-count-to插件对数字显示美化

## 前言

仪表盘上的指标如果一下子跳到终值，看起来像静态文案。需要的是从起点滚到终点，读数过程可见。`vue-count-to` 就是做这件事的轻量组件，管理台首页常见的数字翻牌可以靠它完成，不必自己写 `requestAnimationFrame`。

## Vue-count-to介绍

组件没有额外运行时依赖，能自己判断是向上计数还是向下倒数：把 `startVal` 和 `endVal` 对调即可。缓动函数可以换成自己的。它也支持服务端渲染，思路来自 countUp.js，包名是 `vue-count-to`。

## 依赖安装

```bash
npm install vue-count-to
```

或：

```bash
yarn add vue-count-to
```

## 使用

局部注册即可，不必挂到全局。

```js
import countTo from 'vue-count-to'
```

```js
components: { countTo }
```

下面让「本周完成单」从 0 滚到 128，时长两秒：

```html
<template>
  <countTo :startVal="from" :endVal="to" :duration="2000" :separator="','" />
</template>

<script>
import countTo from 'vue-count-to'
export default {
  components: { countTo },
  data() {
    return { from: 0, to: 128 }
  }
}
</script>
```

终值来自接口时，先给 `to` 一个 0，请求回来再赋值。`autoplay` 为真时，起点或终点一变就会重播。

## 配置

| Property | Description | type | default |
| --- | --- | --- | --- |
| startVal | 起点 | Number | 0 |
| endVal | 终点 | Number | 2017 |
| duration | 动画毫秒数 | Number | 3000 |
| autoplay | 是否自动开始 | Boolean | true |
| decimals | 保留的小数位 | Number | 0 |
| decimal | 小数点符号 | String | . |
| separator | 千分位符号 | String | , |
| prefix | 前缀，例如货币符号 | String | '' |
| suffix | 后缀，例如单位 | String | '' |
| useEasing | 是否使用缓动 | Boolean | true |
| easingFn | 自定义缓动 | Function | — |

`autoplay` 为真时，`startVal` 或 `endVal` 变化会重新播放。

| 方法 | 作用 |
| --- | --- |
| mountedCallback | 挂载完成后的回调 |
| start | 手动开始 |
| pause | 暂停 |
| reset | 回到起点 |

金额不要用这个组件做精确计算，它只负责展示。真实合计仍以接口返回的数字为准，动画只是把同一个数字播出来。
