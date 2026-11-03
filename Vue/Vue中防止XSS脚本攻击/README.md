# Vue中防止XSS脚本攻击

评论要展示表情图片，于是把 `v-text` 换成 `v-html`。标签能渲染了，访客也可以塞脚本。跨站脚本的目标是正在看这条评论的人，不是服务器。

## XSS脚本攻击

攻击者在评论框写入一段可点击的标记，其他读者一点就在自己的页面里执行。例如：

```html
<a href="javascript:void(0)" onclick="alert('bad')">点这里</a>
```

页面上会多出一个链接。若把循环写死成不停 `alert`，浏览器会被弹窗占满，只能强关标签页。表情需求让我们不能一刀切掉所有标签，所以要消毒而不是退回纯文本。

## 解决XSS脚本攻击

安装 `xss`，它按白名单剔除标记和属性。

```bash
npm install xss --save
```

在入口挂到原型上，模板里少写一遍导入：

```js
import xss from 'xss'

Object.defineProperty(Vue.prototype, '$sanitize', {
  value: xss
})
```

渲染处包一层：

```html
<div class="reply-body" v-html="$sanitize(row.body)"></div>
```

默认白名单很严，表情用的 `img` 或带 class 的 `span` 会被剥掉，只剩文字。把允许的标签和属性写进配置，其余仍丢弃：

```js
data() {
  return {
    xssOptions: {
      whiteList: {
        a: ['href', 'title', 'target'],
        span: ['class'],
        img: ['src', 'alt']
      }
    }
  }
}
```

```html
<div class="reply-body" v-html="$sanitize(row.body, xssOptions)"></div>
```

`href` 仍可能写成 `javascript:` 协议。若链接只允许站内，再在白名单之外加一条自定义过滤，丢掉非 `http` / `https` 的地址。不要把 `onerror`、`onclick` 放进属性白名单，表情不需要它们。
