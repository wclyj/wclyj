# 使用Axios拦截器携带token以及跳转错误页面

## 前言

Axios 基于 Promise，浏览器和 Node 都能用。业务里通常包一层实例，而不是到处 `axios.get`。拦截器分两类：请求拦截器在发出去之前改配置，适合塞登录凭证；响应拦截器看业务码或 HTTP 状态，决定把数据交回调用方，还是跳到错误页。

## 创建Axios对象

`baseURL` 留空时走当前站点；超时给一个明确上限，避免请求一直挂着。

```js
import axios from 'axios'

const http = axios.create({
  baseURL: '',
  timeout: 8000
})
```

## Request拦截器

凭证放在自定义头里。这里用 `sessionStorage` 演示，项目里换成自己的读取函数即可。没有凭证时不要写空字符串，否则后端会把空头当成非法令牌。

```js
http.interceptors.request.use(
  (config) => {
    const ticket = sessionStorage.getItem('accessTicket')
    if (ticket) {
      config.headers['X-Access-Ticket'] = ticket
    }
    return config
  },
  (err) => {
    console.warn(err)
    return Promise.reject(err)
  }
)
```

请求拦截器的失败分支必须 `return Promise.reject`，只调用 `Promise.reject` 而不返回，调用方的 `catch` 接不住。

## Response拦截器

下面按业务码分流：成功码原样返回；未登录或参数不合法跳到缺省页；服务端故障交给调用方 `catch`。网络层超时、断网走错误回调，跳到超时提示页。

```js
import router from '@/router'

http.interceptors.response.use(
  (response) => {
    const body = response.data
    if (body.code === 'ok' || body.code === 'fail') {
      return body
    }
    if (body.code === 401 || body.code === 422) {
      console.warn('业务拒绝', body)
      router.push({ name: 'missing' })
      return body
    }
    if (body.code === 500) {
      return Promise.reject(new Error('server'))
    }
    return body
  },
  () => {
    router.push({ name: 'offline' })
    return Promise.reject(new Error('network'))
  }
)

export default http
```

状态码和路由名按自己的约定改。不要在拦截器里又弹一次全局消息、页面上再弹一次，否则一次失败会出现两层提示。
