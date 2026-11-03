# Vue中对数组变化监听

## 前言

列表页经常要改数组某一项，例如把「是否展开回复」从 0 改成 1。按下标直接赋值，数组里的值变了，视图却不动，`watch` 开了 `deep` 也收不到。

```html
<script>
export default {
  data() {
    return {
      threads: [],
      opened: [0, 0, 0, 0]
    }
  },
  methods: {
    toggle(index) {
      this.opened[index] = 1
    }
  }
}
</script>
```

Vue 2 无法侦测「按下标替换」和「改 length」。下面这种监听同样不会触发：

```html
<script>
export default {
  data() {
    return {
      threads: [],
      opened: [0, 0, 0, 0]
    }
  },
  watch: {
    opened: {
      handler() {},
      deep: true
    }
  },
  methods: {
    toggle(index) {
      this.opened[index] = 1
    }
  }
}
</script>
```

要用 Vue 能拦截的写法通知变更。两种都可行：

```js
Vue.set(vm.opened, index, next)
vm.opened.splice(index, 1, next)
```

组件里对应 `this.$set`。同一下标不必两种都调用，选一种即可。

```html
<script>
export default {
  data() {
    return {
      threads: [],
      opened: [0, 0, 0, 0]
    }
  },
  methods: {
    toggle(index) {
      this.$set(this.opened, index, 1)
    }
  }
}
</script>
```

`splice` 会替换那一位并返回被删掉的旧值，适合需要旧值的场景。Vue 3 用 Proxy，按下标赋值已经能触发更新，但这套笔记若仍跑在 Vue 2，不要依赖那个行为。另外，给对象新增一个事先没有的字段，同样要走 `this.$set(obj, key, value)`，和数组是同一类限制。
