# Vue如何制作一个评论模块

## 前言

楼中楼不能靠把每一层写成独立组件再手动嵌三遍，回复深度不固定。Ant Design Vue 的 Comment 提供了头像、作者、操作区这几个插槽，见[评论组件](https://www.antdv.com/components/comment-cn/)。它本身只有一层，博客需要的是：任意层回复、点某一条才出现输入框、提交后立刻出现在对应楼层下面。

下面拆成两个组件：`ThreadList` 负责递归渲染，`ReplyPad` 负责输入。数据放进 Vuex，因为递归组件里 `$emit` 只能通知直接父级，跨多层时事件到不了页面根。

## 封装CommentList组件

单条评论渲染完，再用同一个组件去渲染它的 `children`。组件要有 `name`，模板里才能写自己的标签。没有 `name` 时，文件名并不会自动变成可递归的标签。

```html
<template>
  <div>
    <a-comment v-for="item in nodes" :key="item.id">
      <span slot="actions" @click="openPad(item.id)">回复</span>
      <a slot="author">{{ item.author }}</a>
      <a-avatar slot="avatar" :src="item.face" :alt="item.author" />
      <p slot="content">{{ item.text }}</p>
      <thread-list :nodes="item.children" />
    </a-comment>
  </div>
</template>
<script>
export default {
  name: 'ThreadList',
  props: ['nodes']
}
</script>
```

`name` 写成 `ThreadList` 后，模板里的 `thread-list` 才会解析到自己。只在 `components` 里注册别的组件、漏掉 `name`，递归那一层会报未知组件。

## 在父组件中调用该CommentList组件

演示数据做成树，子回复放在 `children`，叶子节点给空数组，递归才有终点。

```js
threads: [
  {
    id: 'n1',
    author: '何清',
    face: 'https://picsum.photos/seed/n1/64',
    text: '样品 A 的曲线偏高',
    children: [
      {
        id: 'n2',
        author: '宋晚',
        face: 'https://picsum.photos/seed/n2/64',
        text: '第二遍复测也是这样',
        children: [
          {
            id: 'n3',
            author: '何清',
            face: 'https://picsum.photos/seed/n1/64',
            text: '那先标记为异常批次',
            children: []
          }
        ]
      },
      {
        id: 'n4',
        author: '梁川',
        face: 'https://picsum.photos/seed/n4/64',
        text: '我这边对照组正常',
        children: []
      }
    ]
  },
  {
    id: 'n5',
    author: '宋晚',
    face: 'https://picsum.photos/seed/n2/64',
    text: '下午补一组空白',
    children: []
  }
]
```

父页面只把根数组传下去：

```html
<template>
  <thread-list :nodes="threads" />
</template>
<script>
import ThreadList from '../components/ThreadList'
export default {
  components: { ThreadList },
  data() {
    return { threads: [ /* 上面的树 */ ] }
  }
}
</script>
```

到这一步，缩进已经出来了，但还没有输入框。不要在每一层都请求一次接口，整棵树一次拿完，本地插入新节点后再单独提交那一条。

## 封装CommentBox组件

输入区也用 Comment 的内容插槽，头像用当前登录人，而不是被回复的那一条。取消按钮只在楼中楼出现，页顶的主输入框不需要取消。

```html
<template>
  <a-comment>
    <a-avatar slot="avatar" :src="actor.face" :alt="actor.author" />
    <div slot="content">
      <a-form-item>
        <a-textarea :rows="3" :value="draft" @change="onType" />
      </a-form-item>
      <a-form-item>
        <a-button type="primary" :loading="sending" @click="onSend">送出</a-button>
        <a-button v-if="allowClose" style="margin-left: 8px" @click="onClose">收起</a-button>
      </a-form-item>
    </div>
  </a-comment>
</template>
<script>
export default {
  props: {
    actor: { type: Object, required: true },
    targetId: { type: [String, Number], default: 0 },
    allowClose: { type: Boolean, default: true }
  },
  data() {
    return { draft: '', sending: false }
  },
  methods: {
    onType(event) {
      this.draft = event.target.value
    },
    onSend() {
      const text = this.draft.trim()
      if (!text) return
      this.$emit('commit', {
        parentId: this.targetId,
        author: this.actor.author,
        face: this.actor.face,
        text,
        children: []
      })
      this.draft = ''
    },
    onClose() {
      this.draft = ''
      this.$emit('close', this.targetId)
    }
  }
}
</script>
```

`ThreadList` 里每条评论下面放一个默认隐藏的输入区。点「回复」时先把所有输入区藏起来，再只显示被点中的那一个，避免一屏同时展开好几个框。

```html
<reply-pad
  class="reply-pad"
  :id="'pad-' + item.id"
  :actor="actor"
  :target-id="item.id"
  @commit="onCommit"
  @close="onClose"
/>
```

```js
openPad(id) {
  const pads = document.getElementsByClassName('reply-pad')
  Array.from(pads).forEach((node) => { node.style.display = 'none' })
  document.getElementById('pad-' + id).style.display = 'block'
}
```

```css
.reply-pad { display: none; }
```

用 class 批量隐藏是因为输入区散落在递归出来的多层 DOM 里，单靠一个布尔数组按下标对齐很容易和列表错位。id 必须唯一，新评论的 id 用 store 里的递增序号，不能用数组下标。

## 引入Vuex

楼中楼提交后要插到某一条的 `children` 里。若靠 `$emit` 逐层往上抛，每加一层都要记得再转发，漏一层数据就丢了。把整棵树放进 store，任何一层都能读到同一份引用，改完再提交回去，视图会从根重绘。

`store/types.js`：

```js
export const REPLACE_THREADS = 'replaceThreads'
export const NEXT_ID = 'nextId'
```

`store/board.js`：

```js
import { REPLACE_THREADS, NEXT_ID } from './types'

const board = {
  state: {
    threads: [],
    serial: 100
  },
  getters: {},
  mutations: {
    [REPLACE_THREADS](state, threads) {
      state.threads = threads
    },
    [NEXT_ID](state) {
      state.serial += 1
    }
  },
  actions: {}
}
export default board
```

`store/index.js`：

```js
import Vue from 'vue'
import Vuex from 'vuex'
import board from './board'

Vue.use(Vuex)

export default new Vuex.Store({
  modules: { board }
})
```

带回复框的列表组件。提交时如果 `parentId` 为 0，说明是顶层发言，直接追加到根数组；否则递归找到那条父评论，推进它的 `children`。

```html
<template>
  <div>
    <a-comment v-for="item in nodes" :key="item.id">
      <span slot="actions" @click="openPad(item.id)">回复</span>
      <a slot="author">{{ item.author }}</a>
      <a-avatar slot="avatar" :src="item.face" :alt="item.author" />
      <p slot="content">{{ item.text }}</p>
      <reply-pad
        class="reply-pad"
        :id="'pad-' + item.id"
        :actor="actor"
        :target-id="item.id"
        @commit="onCommit"
        @close="onClose"
      />
      <thread-list :nodes="item.children" />
    </a-comment>
  </div>
</template>
<script>
import { mapMutations } from 'vuex'
import ReplyPad from './ReplyPad'

export default {
  name: 'ThreadList',
  components: { ReplyPad },
  props: ['nodes'],
  data() {
    return {
      actor: {
        author: '值班员',
        face: 'https://picsum.photos/seed/me/64'
      }
    }
  },
  methods: {
    ...mapMutations(['replaceThreads', 'nextId']),
    openPad(id) {
      const pads = document.getElementsByClassName('reply-pad')
      Array.from(pads).forEach((node) => { node.style.display = 'none' })
      const hit = document.getElementById('pad-' + id)
      if (hit) hit.style.display = 'block'
    },
    onClose(id) {
      const hit = document.getElementById('pad-' + id)
      if (hit) hit.style.display = 'none'
    },
    onCommit(payload) {
      const tree = this.$store.state.board.threads
      payload.id = 'n' + this.$store.state.board.serial
      if (!payload.parentId) {
        tree.push(payload)
      } else {
        this.attach(tree, payload.parentId, payload)
        this.onClose(payload.parentId)
      }
      this.replaceThreads(tree.slice())
      this.nextId()
    },
    attach(list, parentId, payload) {
      if (!list || !list.length) return
      list.forEach((item) => {
        if (item.id === parentId) item.children.push(payload)
        else this.attach(item.children, parentId, payload)
      })
    }
  }
}
</script>
<style>
.reply-pad { display: none; }
</style>
```

`replaceThreads(tree.slice())` 换了一个新数组引用，列表才会刷新。若 mutation 收到的是同一引用，Vuex 不一定能通知到依赖这棵树的计算属性。递归插入时改的是节点上的 `children`，父数组引用没变，所以要切片再提交一次。

页面根组件负责两件事：顶上放一个不能收起的输入框，用来发一级评论；挂载时把初始树写入 store。列表组件读的是 store，不是父组件的本地 `data`，两边才是同一份。

```html
<template>
  <div class="board">
    <reply-pad
      :actor="actor"
      :target-id="0"
      :allow-close="false"
      @commit="onRoot"
    />
    <thread-list :nodes="threads" />
  </div>
</template>
<script>
import { mapState, mapMutations } from 'vuex'
import ThreadList from '../components/ThreadList'
import ReplyPad from '../components/ReplyPad'

export default {
  components: { ThreadList, ReplyPad },
  data() {
    return {
      actor: {
        author: '值班员',
        face: 'https://picsum.photos/seed/me/64'
      },
      seed: [
        {
          id: 'n1',
          author: '何清',
          face: 'https://picsum.photos/seed/n1/64',
          text: '样品 A 的曲线偏高',
          children: [
            {
              id: 'n2',
              author: '宋晚',
              face: 'https://picsum.photos/seed/n2/64',
              text: '第二遍复测也是这样',
              children: []
            }
          ]
        }
      ]
    }
  },
  computed: {
    ...mapState({
      threads: (state) => state.board.threads
    })
  },
  mounted() {
    this.replaceThreads(this.seed)
  },
  methods: {
    ...mapMutations(['replaceThreads', 'nextId']),
    onRoot(payload) {
      const tree = this.$store.state.board.threads.slice()
      payload.id = 'n' + this.$store.state.board.serial
      tree.push(payload)
      this.replaceThreads(tree)
      this.nextId()
    }
  }
}
</script>
<style>
.board { width: 640px; margin: 0 auto; }
</style>
```

一级评论的 `parentId` 固定为 0，和楼中楼共用 `onCommit` 的前半段也行，这里拆开是为了让页顶输入框不依赖列表内部的 DOM id。接口保存应发生在更新 store 之前：请求失败就不要 `push`，否则刷新后本地多出来的那条会消失，读者会以为已经发出去了。

表情和链接若要按 HTML 渲染，先做消毒再绑定，不要把 `text` 直接丢进 `v-html`。消毒写法见「防止 XSS」那篇，白名单里不要放事件属性。

把一次楼中楼提交按时间顺序拆开，方便对照 store 里的树。假设初始树只有 `n1`，它的 `children` 里有 `n2`。当前 `serial` 是 100。

1. 用户点 `n2` 的「回复」。`openPad` 把所有 `.reply-pad` 设成不显示，再把 `pad-n2` 设成显示。这一步不碰 store。
2. 输入框发出 `commit`，载荷里 `parentId` 是 `n2`，正文是新句子，还没有 `id`。
3. `onCommit` 读 `state.board.serial`，把 id 写成 `n100`。
4. `attach` 从根走到 `n2`，把新节点推进 `n2.children`。叶子的 `children` 必须一开始就是数组，不能是 `null`，否则 `push` 会抛错，整次提交中断。
5. `replaceThreads(tree.slice())` 换掉根数组引用。列表的 `nodes` 来自计算属性，引用变了才会重绘。只改深层 `children`、不换根引用时，有的页面不会刷新。
6. `nextId` 把 `serial` 加成 101。下一条新评论才不会和 `n100` 撞 id。
7. `onClose` 藏起 `pad-n2`。若先销毁再改数据，递归重绘可能来不及，输入框会留在新楼层下面。

顶层发言不走 `attach`。页顶的 `ReplyPad` 把 `targetId` 固定传 0，`onRoot` 只对根数组 `push`。不要让顶层输入框也去调 `openPad`，它没有对应的楼层 id，`getElementById` 会拿到空。

初始树建议在接口返回后再写入 store，而不是在 `data` 里写死一份又在 `mounted` 再拷一次。本地演示可以保留种子数据，接上接口后删掉种子，避免第一帧闪出假评论：

```js
async mounted() {
  const list = await this.$http.get('/threads', { params: { topicId: this.topicId } })
  this.replaceThreads(normalize(list))
}

function normalize(list) {
  return (list || []).map((item) => ({
    id: item.id,
    author: item.author,
    face: item.face || '',
    text: item.text || '',
    children: normalize(item.children)
  }))
}
```

`normalize` 保证每一层都有 `children` 数组。后端用 `null` 表示没有回复时，不归一化就会在 `attach` 里失败。

并发上，两人同时回复同一条时，后到达的请求应基于服务端返回的新树覆盖本地，而不是各自 `push` 后再以本地为准。简单做法：提交接口返回「插入后的那一条」，本地按返回值插入；若返回 409，拉一次全量树覆盖 store。不要在失败时把已经 `push` 的节点留着，刷新后它会消失，作者会以为丢了内容。

递归组件的 `name` 和文件名无关。打包后组件名若被压缩，模板里的标签仍按 `name` 选项解析，一般不受影响。若关闭了运行时编译、模板被预编译成渲染函数，自引用要在 `components` 里显式登记自己，否则预编译阶段不知道标签指向谁：

```js
components: { ReplyPad, ThreadList: () => import('./ThreadList.vue') }
```

同步导入自己会造成循环依赖。上面用异步组件绕开。若构建报循环，把「单条」和「列表」拆成两个文件：列表只循环，单条只负责一条和它的子列表。子列表再引用列表组件。这样没有文件自己 import 自己。

展示层不要在递归里发请求。每一层各打一次「取子回复」会把一棵树变成 N 次往返，而且加载顺序不确定，楼层会跳动。一次取完整棵树，本地插入，保存接口只提交新的那一条。删除则走相反方向：找到父节点，从 `children` 里滤掉目标 id，再 `replaceThreads`。过滤时要拷贝每一层，避免一边遍历一边改原数组：

```js
dropNode(list, id) {
  return list
    .filter((item) => item.id !== id)
    .map((item) => ({ ...item, children: this.dropNode(item.children || [], id) }))
}
```

分页只适合根评论。子回复一旦分页，递归组件要额外知道这一层是否还有下一页，复杂度和收益不成比例。根评论翻页时替换 store 里的根数组，已展开的输入框会丢掉。翻页前把未提交的草稿存到组件本地，不要只留在已经被销毁的文本框里。

楼层缩进用子组件自己的左边距，不要用全局选择器按嵌套层数去算。递归深度不固定，选择器写到第三层就停了，再深的回复会贴回左边，看起来像新的根评论。每一层包一层 `div`，统一 `padding-left: 24px`，深度再增加也只是多包一层。

作者名和头像不要从评论正文里解析。正文可以被改成任何字符串，身份字段应来自接口的用户对象。本地演示把 `actor` 写死，接入登录后改成读会话里的昵称和头像，提交载荷仍只带用户 id，昵称由服务端填充，避免伪造别人的名字。

空树要有占位。`threads` 为 `[]` 时递归组件什么都不画，读者会以为模块坏了。父页面在长度为 0 时显示一句「还没有回复」，有数据再挂 `ThreadList`。不要把占位写进递归组件，否则每一层空的 `children` 都会出现这句话。

提交按钮在请求未完成时要禁用。连点会把同一句推进两次，树里出现两条只差 id 的重复回复。`sending` 从点击开始到接口返回再放开。接口返回的 id 若和本地预占的 `serial` 不同，以接口为准改掉节点 id，否则下一次回复会挂到一个服务端不认识的父 id 上。草稿框失焦时把未发送文本记在会话存储里，按主题 id 分键，刷新页面还能恢复。发送成功后立刻删掉该键，避免下一篇主题读到上一篇的半句话。本地序号只用于乐观更新。刷新之后一律以接口返回的整棵树为准，不要把本地临时节点再合并进去。合并规则一复杂，就会留下没有父节点的回复，刷新后也找不到该挂在哪一层。
