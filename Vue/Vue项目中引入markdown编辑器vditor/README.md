# Vue项目中引入markdown编辑器vditor

## 前言

Vditor 的所见即所得和分屏预览接近桌面 Markdown 编辑器，适合写文章，而不是只做评论框。官方演示在 `https://vditor.b3log.org/demo/vue.html`，仓库为 [Vditor](https://github.com/Vanessa219/vditor)。下面把它收成一个可复用组件，高度由父级传入，图片走自己的上传接口。

## 安装依赖

```bash
npm install vditor --save
```

样式随包走，由组件在脚本里引入，不必再在 `index.html` 挂一份不带版本号的 CDN。生产环境若坚持外链，地址要锁版本，例如 `vditor@3.10.6`，否则上游改了工具栏结构，本地配置会对不上。

## 封装成组件

新建 `components/MdPad/index.vue`。实例在 `mounted` 里创建，因为要有真实 DOM id。上传回调把接口返回的地址插成图片语法，而不是整段替换正文。

```html
<template>
  <div class="md-pad" v-loading="booting">
    <div id="md-pad-root" :style="{ height: height + 'px' }"></div>
  </div>
</template>

<script>
import Vditor from 'vditor'
import 'vditor/dist/index.css'
import { readTicket } from '@/utils/auth'

export default {
  name: 'MdPad',
  props: { height: { type: Number, default: 420 } },
  data() {
    return { booting: true, editor: null }
  },
  mounted() {
    this.boot()
    this.$nextTick(() => { this.booting = false })
  },
  beforeDestroy() {
    if (this.editor) this.editor.destroy()
  },
  methods: {
    boot() {
      this.editor = new Vditor('md-pad-root', {
        height: this.height,
        mode: 'ir',
        outline: { enable: true, position: 'right' },
        cache: { enable: false },
        upload: {
          max: 4 * 1024 * 1024,
          handler: (files) => this.pushImage(files)
        }
      })
    },
    pushImage(files) {
      const form = new FormData()
      Array.from(files).forEach((file) => form.append('media', file))
      const xhr = new XMLHttpRequest()
      xhr.open('POST', process.env.VUE_APP_MEDIA + '/notes/images')
      xhr.setRequestHeader('X-Access-Ticket', readTicket())
      xhr.onload = () => {
        if (xhr.status !== 200) {
          this.$message.error('图片未上传')
          return
        }
        const body = JSON.parse(xhr.responseText)
        this.editor.insertValue('![' + body.name + '](' + body.href + ')')
      }
      xhr.send(form)
    },
    readHtml() {
      return this.editor ? this.editor.getHTML() : ''
    },
    writeMarkdown(text) {
      this.$nextTick(() => {
        if (this.editor) this.editor.setValue(text || '')
      })
    }
  }
}
</script>

<style>
.md-pad { background: #fff; }
</style>
```

默认缓存会把内容写进 `localStorage`，多篇文章共用一个 id 时会串稿。上面关掉了缓存，切换文章时由父组件调用 `writeMarkdown`。

父页面：

```js
import MdPad from '@/components/MdPad'
```

```js
components: { MdPad }
```

```html
<MdPad ref="pad" :height="480" />
```

保存时调用 `this.$refs.pad.readHtml()`，不要去读编辑器内部的 DOM。更多工具栏项见仓库文档，按需往构造参数里加，不要把演示页的配置原样搬过来，里面包含演示站自己的上传域名。

## 效果

即时渲染模式下，左侧改一句，右侧大纲会跟着标题变。图片超过 `max` 时组件会直接拒绝，不会打到接口。移动端宽度不足时把 `outline.enable` 关掉，否则大纲会把编辑区挤没。

编辑器在 `v-if` 里时，销毁和重建要成对。只把外层 `display` 设成 `none` 再显示，工具栏宽度会按隐藏时的 0 计算，恢复后出现一块空白。对话框里使用时，等对话框打开的下一拍再 `boot`，关闭时 `destroy`，不要复用一个已经量过 0 宽度的实例。

`getHTML()` 和 `getValue()` 不是同一个结果。前者是 HTML，适合直接入库给详情页；后者是 Markdown 源文，适合再给别的编辑器。保存接口要定一种，不要这一次存 HTML、下一次存 Markdown，回显时会把标签当源文显示出来。
