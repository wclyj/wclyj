# Vue中input框自动聚焦

## 前言

登录框绑了回车提交，但第一次进页面焦点停在页面空白处，回车不会进输入框的监听。浏览器记住密码之后，账号密码框往往已经有值，用户期望按回车就能登录，所以进入页面时要把焦点放到账号框。

## 解决方案

给目标输入框一个 `ref`，等 DOM 就绪再调用组件的 `focus`。

```html
<el-input
  v-model="form.account"
  ref="accountBox"
  name="account"
  type="text"
  autocomplete="username"
  placeholder="账号"
  @keyup.enter.native="submitLogin"
/>
```

`created` 执行时真实输入节点还没挂上，直接 `this.$refs.accountBox.focus()` 拿不到实例。`$nextTick` 会把回调推迟到这一轮 DOM 更新之后：

```js
created() {
  this.$nextTick(() => {
    this.$refs.accountBox.focus()
  })
}
```

`mounted` 执行时子组件已经挂载，可以不包 `$nextTick`：

```js
mounted() {
  this.$refs.accountBox.focus()
}
```

Element 的 `el-input` 把 `focus` 做在组件上，调用的是组件方法，不是原生 DOM。若换成原生 `<input ref="accountBox">`，则要写成 `this.$refs.accountBox.focus()` 仍然成立，因为原生元素本身就有这个方法。不要在 `focus` 之外再调一次 `select()`，否则浏览器自动填充的密码可能被整段选中，用户一打字就清空了。
