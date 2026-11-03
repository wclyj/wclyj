# Vue项目引入侧边导航栏

## 前言

长文章需要一份跟着滚动的目录，点某一节就跳到对应标题。`vue-side-catalog` 能扫描容器里的标题并生成侧栏，适合文档页，不适合本身已经有锚点组件的后台布局。

## 安装

### 官网

仓库地址：[vue-side-catalog](https://github.com/yaowei9363/vue-side-catalog)。克隆后在该仓库里先跑通示例，确认交互再抄组件，避免把未完成的分支拷进业务仓库。

```bash
npm install
npm run serve
```

示例默认在 `http://localhost:8080/`。把源码里的目录组件拷到业务工程的 `components/DocOutline`，入口文件改名为 `index.vue`，这样引用路径稳定。

### 安装依赖

组件内部用了节流和防抖，业务工程要单独安装，不能假设克隆仓库的 `node_modules` 会跟着拷贝过来。

```bash
npm install lodash.debounce --save
npm install lodash.throttle --save
```

### 使用

```js
import DocOutline from '../components/DocOutline'
```

`containerElementSelector` 指向正文容器。目录只扫描这个节点内部的标题。

```html
<template>
  <div id="app">
    <article class="sheet" v-html="sheetHtml"></article>
    <doc-outline v-bind="outlineProps" />
  </div>
</template>
<script>
import DocOutline from '../components/DocOutline'
export default {
  components: { DocOutline },
  data() {
    return {
      sheetHtml: '',
      outlineProps: {
        containerElementSelector: '.sheet'
      }
    }
  },
  mounted() {
    setTimeout(() => {
      this.sheetHtml = '<h2>材料</h2><h3>步骤</h3><p>按顺序加入。</p>'
    }, 400)
  }
}
</script>
```

标题层级要连续。从 `h2` 直接跳到 `h4` 时，侧栏缩进会空一层，点击偏移也会偏。

## 注意

正文不要写在 `created` 里。那时节点还没挂上，组件扫描不到标题。放在 `mounted`，或在异步赋值之后用 `$nextTick` 再通知目录刷新。若正文来自接口，等 `v-html` 更新完再滚动，否则目录是空的，过一秒才出现，锚点高度也是旧的。
