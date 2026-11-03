## VueX学习

用一个加减计数器把 Vuex 的四块职责走一遍：状态放哪、怎么读、谁允许改、异步步骤放哪。示例是场次得分 `tally`，不是把接口数据散落在各个页面的 `data` 里。

### 使用vue-cli创建vue项目

```bash
npm install -g @vue/cli-init
vue init webpack-simple score-desk
cd score-desk
npm install
npm install vuex@3
```

`webpack-simple` 模板没有 Vuex。Vue 2 配 Vuex 3，不要装 Vuex 4，那是给 Vue 3 的。

### 修改main.js

先让插件生效，再创建一个 store。store 在应用启动时只有一份，页面通过 `this.$store` 拿到它。

```js
import Vuex from 'vuex'

Vue.use(Vuex)
```

四块的分工：

- `state` 放真正的数据，这里是当前得分。
- `getters` 像只读的派生值，适合拼文案或做过滤，不要在里面改 state。
- `mutations` 是唯一允许改 state 的地方，必须是同步函数。
- `actions` 放多步流程。需要改状态时，它不自己赋值，而是 `commit` 某个 mutation。

```js
const store = new Vuex.Store({
  state: {
    tally: 0
  },
  getters: {
    tallyText(state) {
      return '当前得分 ' + state.tally
    }
  },
  mutations: {
    addOne(state) {
      state.tally += 1
    },
    subOne(state) {
      state.tally -= 1
    }
  },
  actions: {
    rise(context) {
      context.commit('addOne')
    },
    fall(context) {
      context.commit('subOne')
    }
  }
})
```

把 store 交给根实例，子组件才能注入到：

```js
new Vue({
  el: '#app',
  render: (h) => h(App),
  store
})
```

入口文件合在一起是：

```js
import Vue from 'vue'
import App from './App.vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.Store({
  state: { tally: 0 },
  getters: {
    tallyText(state) {
      return '当前得分 ' + state.tally
    }
  },
  mutations: {
    addOne(state) { state.tally += 1 },
    subOne(state) { state.tally -= 1 }
  },
  actions: {
    rise(context) { context.commit('addOne') },
    fall(context) { context.commit('subOne') }
  }
})

new Vue({
  el: '#app',
  render: (h) => h(App),
  store
})
```

### 修改App.vue页面

页面不直接改 `tally`，而是用辅助函数把 store 上的字段和方法映射到计算属性和方法里。

```js
import { mapState, mapGetters, mapMutations, mapActions } from 'vuex'
```

计算属性里展开 state 和 getter。映射回来的名字和 store 里一致，模板才能写 `tally`、`tallyText`。

```js
computed: {
  ...mapState(['tally']),
  ...mapGetters(['tallyText'])
}
```

改值有两条等价路径。一条是映射 mutation，再在点击里调用：

```js
methods: {
  ...mapMutations(['addOne', 'subOne'])
}
```

另一条不映射，点击时自己提交：

```js
methods: {
  onRise() { this.$store.commit('addOne') },
  onFall() { this.$store.commit('subOne') }
}
```

日常更常见的是走 action。action 以后可以在 `commit` 前加校验或请求，页面不用改：

```js
methods: {
  ...mapActions(['rise', 'fall']),
  onRise() { this.rise() },
  onFall() { this.fall() }
}
```

页面完整示例：

```html
<template>
  <div id="app">
    <p>{{ tally }}</p>
    <p>{{ tallyText }}</p>
    <button type="button" @click="onRise">加一</button>
    <button type="button" @click="onFall">减一</button>
  </div>
</template>

<script>
import { mapState, mapGetters, mapActions } from 'vuex'

export default {
  name: 'app',
  computed: {
    ...mapState(['tally']),
    ...mapGetters(['tallyText'])
  },
  methods: {
    ...mapActions(['rise', 'fall']),
    onRise() { this.rise() },
    onFall() { this.fall() }
  }
}
</script>
```

### 运行的结果

装好 Vue Devtools 后重新打开浏览器，在调试面板的 Vuex 页能看到 `tally` 的每次变化。点「加一」应变为 1，点「减一」再回到 0。若面板没有 Vuex 页，多半是生产构建把 devtools 关掉了，开发模式再看一次。

### 小结

- 同步、一句就能改完的状态更新放进 mutation。带判断、要等接口的步骤放进 action，action 里再 `commit`。
- 和界面相关的跳转、弹消息不要写进 action，action 只处理状态怎么变。页面拿到新状态后自己决定提示什么。

### mutations和actions自定义传参

一次加 N 分时，mutation 的第二个参数就是这次的载荷。action 同样接收第二个参数，再原样交给 `commit`。

```js
mutations: {
  addOne(state) { state.tally += 1 },
  subOne(state) { state.tally -= 1 },
  addBy(state, step) { state.tally += step }
},
actions: {
  rise(context) { context.commit('addOne') },
  riseBy(context, step) { context.commit('addBy', step) },
  fall(context) { context.commit('subOne') }
}
```

页面上多一个按钮，把步长传进去。步长来自输入框时先转成数字，避免把字符串拼进得分。

```js
methods: {
  ...mapActions(['rise', 'fall', 'riseBy']),
  onRise() { this.rise() },
  onFall() { this.fall() },
  onRiseBy() { this.riseBy(5) }
}
```

带步长的入口：

```js
import Vue from 'vue'
import App from './App.vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.Store({
  state: { tally: 0 },
  getters: {},
  mutations: {
    addOne(state) { state.tally += 1 },
    subOne(state) { state.tally -= 1 },
    addBy(state, step) { state.tally += Number(step) || 0 }
  },
  actions: {
    rise(context) { context.commit('addOne') },
    riseBy(context, step) { context.commit('addBy', step) },
    fall(context) { context.commit('subOne') }
  }
})

new Vue({ el: '#app', render: (h) => h(App), store })
```

对应页面：

```html
<template>
  <div id="app">
    <p>{{ tally }}</p>
    <button type="button" @click="onRise">加一</button>
    <button type="button" @click="onFall">减一</button>
    <button type="button" @click="onRiseBy">加五分</button>
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex'

export default {
  computed: {
    ...mapState(['tally'])
  },
  methods: {
    ...mapActions(['rise', 'fall', 'riseBy']),
    onRise() { this.rise() },
    onFall() { this.fall() },
    onRiseBy() { this.riseBy(5) }
  }
}
</script>
```

再点一次「加五分」，devtools 里 `tally` 应增加 5。若没有任何变化，先看 action 名字是否和 `mapActions` 数组里的字符串一致，拼写不一致时点击是静默的。

### 将main.js中的store提取出来

store 继续堆在入口里，登录信息和得分会缠在一起。拆成目录后，入口只负责引入。

```text
src/store/
  index.js
  types.js
  score.js
  session.js
```

`types.js` 把 mutation 名字收成常量，提交时用常量，避免手写字符串对不上：

```js
export const ADD_ONE = 'addOne'
export const SUB_ONE = 'subOne'
export const ADD_BY = 'addBy'
```

`score.js` 是原来的得分模块：

```js
import { ADD_ONE, SUB_ONE, ADD_BY } from './types'

const score = {
  state: { tally: 0 },
  getters: {
    tallyText(state) {
      return '当前得分 ' + state.tally
    }
  },
  mutations: {
    [ADD_ONE](state) { state.tally += 1 },
    [SUB_ONE](state) { state.tally -= 1 },
    [ADD_BY](state, step) { state.tally += Number(step) || 0 }
  },
  actions: {
    rise(context) { context.commit(ADD_ONE) },
    riseBy(context, step) { context.commit(ADD_BY, step) },
    fall(context) { context.commit(SUB_ONE) }
  }
}

export default score
```

登录态单独一个模块，哪怕暂时是空的，也别塞进得分文件：

```js
const session = {
  state: { profile: null },
  getters: {},
  mutations: {},
  actions: {}
}
export default session
```

`index.js` 用 `modules` 把它们挂到同一棵树上：

```js
import Vue from 'vue'
import Vuex from 'vuex'
import score from './score'
import session from './session'

Vue.use(Vuex)

export default new Vuex.Store({
  modules: { score, session }
})
```

入口瘦成：

```js
import Vue from 'vue'
import App from './App.vue'
import store from './store'

new Vue({
  el: '#app',
  render: (h) => h(App),
  store
})
```

分模块之后，`tally` 不再位于根 state，而在 `state.score.tally`。`mapState` 要改成函数形式。getter 若没有 `namespaced: true`，仍然注册在全局，名字不能和别的模块撞车。

```html
<template>
  <div id="app">
    <p>{{ tally }}</p>
    <p>{{ tallyText }}</p>
    <button type="button" @click="onRise">加一</button>
    <button type="button" @click="onFall">减一</button>
    <button type="button" @click="onRiseBy">加五分</button>
  </div>
</template>

<script>
import { mapState, mapGetters, mapActions } from 'vuex'
import { ADD_ONE, SUB_ONE } from './store/types'

export default {
  computed: {
    ...mapState({
      tally: (state) => state.score.tally
    }),
    ...mapGetters(['tallyText'])
  },
  methods: {
    ...mapActions(['rise', 'fall', 'riseBy']),
    onRise() { this.rise() },
    onFall() { this.fall() },
    onRiseBy() { this.riseBy(5) }
  }
}
</script>
```

`ADD_ONE` 留在文件里是为了页面若改走 `commit(ADD_ONE)` 时不用再写魔法字符串。开启 `namespaced: true` 后，action 要写成 `score/rise`，`mapActions` 的写法也要带模块前缀。这个计数器先不开命名空间，等模块变多、出现同名 action 时再开。

调试时若按钮点了没反应，按这个顺序看：devtools 里有没有这次 mutation；没有的话是 action 没提交，或名字和 `mapActions` 不一致；有 mutation 但页面数字不动，多半是模板绑的字段还在根 state，分模块后忘了改成 `state.score.tally`。不要在模板里写 `this.$store.state.tally` 这种只在拆模块之前才成立的路径。

另外，mutation 保持同步。若把 `setTimeout` 或接口调用写进 mutation，devtools 的时间旅行会对不上，严格模式也会给出警告。等待放在 action 里，拿到结果再 `commit` 一个只做赋值的 mutation。计数器以后若要和后端对账，就在 `rise` 里先请求，成功后再 `addOne`，失败就不要提交。页面上展示的数字只从计算属性来，不要再在 `data` 里留一份本地副本，两份数字很快就会不一致。
