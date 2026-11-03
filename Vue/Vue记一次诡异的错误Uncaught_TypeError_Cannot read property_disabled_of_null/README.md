前言
--

第三方登录入口一点就抛出 `Uncaught TypeError: Cannot read property 'disabled' of null`。第一反应是按钮上的 `disabled` 写错了：

```html
<el-button type="success" circle disabled>
  <span class="iconfont">&#xe712;</span>
</el-button>
```

把页面上所有 `disabled` 去掉，报错还在。继续往下看，问题出在下拉菜单：未登录时用 `v-if` 把整个菜单拿掉了。

```html
<el-dropdown @command="onPick" class="avatar-menu">
  <span class="el-dropdown-link" @click="goLogin">
    <img v-if="!signedIn" src="../../static/images/guest.png">
    <img v-else-if="profile.avatar" :src="mediaHost + profile.avatar">
    <img v-else src="https://example.com/fallback-avatar.gif">
  </span>
  <el-dropdown-menu slot="dropdown" v-if="signedIn">
    <el-dropdown-item command="home">主页</el-dropdown-item>
    <el-dropdown-item command="signout">退出</el-dropdown-item>
  </el-dropdown-menu>
</el-dropdown>
```

`el-dropdown` 在初始化时会去找菜单项，并读取项上的 `disabled`。`v-if` 为假时菜单根本没有被创建，内部拿到的是 `null`，再读属性就抛了上面的错误。

菜单保留在文档里，只控制每一项显不显。`v-show` 仍会渲染节点，只是加上隐藏样式，组件初始化找得到子节点。

```html
<el-dropdown-menu slot="dropdown">
  <el-dropdown-item command="home" v-show="signedIn">主页</el-dropdown-item>
  <el-dropdown-item command="signout" v-show="signedIn">退出</el-dropdown-item>
</el-dropdown-menu>
```

未登录时菜单是空的，但节点还在，报错消失。不要把 `v-if` 放在 `el-dropdown-menu` 上。若必须按登录态拆成两套结构，就整颗 `el-dropdown` 做 `v-if` / `v-else`，不要只拆掉菜单。
