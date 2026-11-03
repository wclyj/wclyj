前言
--

登录接口把凭证写进 Cookie 后，带 `www` 的地址能看到头像，刷新成不带 `www` 的根域时头像和凭证一起消失。打开调试面板，Cookie 列表也是空的。点回带子域的链接，凭证又出现了。有效期明明写了七天，不是浏览器一关就清。根因是：脚本默认只把 Cookie 记在当前主机名上，`www.example.cn` 和 `example.cn` 互不相通。`a.example.cn` 写下的值，`b.example.cn` 同样读不到。

原理
--

要让同一站点的各个主机共享，必须显式设置 `domain`，并且以点开头，例如 `.example.cn`。这样以该后缀结尾的主机都能带上这枚 Cookie。

常见字段：

*   name：键名，创建后不能改
*   value：键值。含非 ASCII 时先编码；二进制先做 Base64
*   maxAge：寿命，单位秒。正数表示多少秒后过期；负数表示会话 Cookie，关浏览器即丢；0 表示立刻删除。缺省按 -1 处理
*   secure：为真时只在 HTTPS 等加密通道发送
*   path：生效路径。写成 `/sessionWeb/` 时，只有该上下文能读；写成 `/` 时当前域下都可读。结尾要保留斜杠
*   domain：哪些主机可以携带。设成 `.example.cn` 后，凡是以此结尾的主机都能读。开头那个点不能省
*   comment：给人看的用途说明，浏览器展示 Cookie 时可能带上
*   version：0 表示旧的 Netscape 写法，1 表示遵循 RFC 2109

从当前主机推父域，可以先按点拆开，丢掉最左边一段，再拼回去并补上前导点：

```js
const host = location.hostname
const parts = host.split('.')
parts.shift()
const parent = '.' + parts.join('.')
```

本地开发的主机名没有点（例如 `localhost`），不能套这套父域逻辑，否则浏览器会直接丢掉这枚 Cookie。另外，如果父域本身是公共后缀，写入会失败，这时要退回到「点 + 当前完整主机名」。

完整工具如下。写入成功与否，用读回的值比对，而不是假设第一条 `domain` 一定生效。

```js
/** 跨子域共享的 Cookie 读写 */
export function writeSharedCookie(key, raw, days) {
  const encoded = encodeURIComponent(raw)
  let lifetime = ''
  if (typeof days === 'number') {
    const when = new Date(Date.now() + days * 86400000)
    lifetime = '; expires=' + when.toUTCString()
  }

  const host = location.hostname
  if (host.split('.').length < 2) {
    document.cookie = key + '=' + encoded + lifetime + '; path=/'
    return
  }

  const parts = host.split('.')
  parts.shift()
  let scope = '.' + parts.join('.')
  document.cookie = key + '=' + encoded + lifetime + '; path=/; domain=' + scope

  if (readSharedCookie(key) !== raw) {
    scope = '.' + host
    document.cookie = key + '=' + encoded + lifetime + '; path=/; domain=' + scope
  }
}

export function readSharedCookie(key) {
  const rule = new RegExp('(?:^|; )' + key + '=([^;]*)')
  const found = document.cookie.match(rule)
  return found ? decodeURIComponent(found[1]) : null
}

export function dropSharedCookie(key) {
  const current = readSharedCookie(key)
  if (current == null) return
  const expired = new Date(Date.now() - 1000).toUTCString()
  document.cookie = key + '=' + encodeURIComponent(current) + '; expires=' + expired + '; path=/'
}
```

删除时也要带上当初的 `path` 和 `domain`，否则浏览器会当成另一枚 Cookie，页面上看着还在。
