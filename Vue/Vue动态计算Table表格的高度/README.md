前言
--

屏幕高度不一样时，表格若写死像素，矮屏会出现双滚动条，高屏则底部空一块。挂载后用窗口高度减去检索条和顶栏，把剩下的值交给表格的 `height`。

代码
--

先量检索条的实际高度，再扣掉顶栏、页签和分页大约占用的固定值。固定值按自己的布局改，不要照搬。

```js
mounted() {
  const bar = window.getComputedStyle(this.$refs.filterBar).height
  const barHeight = parseInt(bar, 10)
  this.gridHeight = window.innerHeight - barHeight - 220
}
```

模板里表格绑定 `:height="gridHeight"`，检索条加上 `ref="filterBar"`。窗口缩放时这段不会重算，需要再听 `resize`，并在 `beforeDestroy` 里解绑。`getComputedStyle` 在元素尚未渲染时会报错，所以不要放进 `created`。
