前言
--

页头和内容区是两个单文件组件。在页头提交检索词后，内容区要立刻按这个词重拉列表。父子传参走不通，因为它们没有直接的父子关系，只是被布局拼在同一屏。

检索框在顶栏，结果表在主区域。顶栏不该去 `import` 结果表，结果表也不该去读顶栏的私有 `data`。

解决方案
----

用 Vuex 做一块双方都看得到的状态：

*   顶栏在提交时调用 mutation，把检索词写进 store
*   结果表用 `watch` 盯着这段状态，变了就重新请求

具体代码如下
------

store 的拆分方式可以参考同目录的计数器笔记。这里只加一个检索词字段。

```js
import { SET_QUERY } from './mutation-types'

const app = {
  state: {
    query: { text: '', stamp: 0 }
  },
  getters: {},
  mutations: {
    [SET_QUERY](state, query) {
      state.query = query
    }
  },
  actions: {}
}
export default app
```

`stamp` 用时间戳，是为了连续两次输入同一个词时也能触发监听。只存字符串的话，第二次相同内容不会被认为变化。

顶栏引入映射方法：

```js
import { mapMutations } from 'vuex'
```

```js
methods: {
  ...mapMutations(['setQuery']),
  onSearch(text) {
    this.setQuery({ text, stamp: Date.now() })
  }
}
```

结果表监听模块上的字段。模块化之后路径是 `$store.state.app.query`，不是根上的 `query`。

```js
watch: {
  '$store.state.app.query': function (next) {
    this.loadTable(next.text)
  }
}
```

不要在 watch 里再提交同一个 mutation，否则会形成循环。检索请求放在结果表，顶栏只负责把词交出去。
