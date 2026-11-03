# Vue对Element中的el-tag添加@click事件无效

## 前言

评论列表里想点标签跳到原文，于是在 `el-tag` 上写了点击监听，点下去没有反应：

```html
<el-tag type="info" @click="openPost(row.post)">{{ row.post.title }}</el-tag>
```

按钮上同样的写法是有效的，标签上无效。

## 解决方法

Element UI 2 的标签根节点是原生元素，但组件没有把 `click` 声明成组件事件，Vue 会把它当成监听一个不存在的自定义事件。原生 DOM 事件要加 `.native` 修饰符，按钮组件则在内部做了转发，所以不用加。

```html
<el-tag type="info" @click.native="openPost(row.post)">{{ row.post.title }}</el-tag>
```

Vue 3 去掉了 `.native`，监听会落到根节点。若工程以后升级，这个修饰符要删掉，否则编译期会提示无法识别。标签若被 `disabled` 样式挡住指针事件，修饰符加上也点不中，先看样式里有没有 `pointer-events: none`。
