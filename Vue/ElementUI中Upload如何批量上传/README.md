# ElementUI中Upload组件如何批量上传

## 前言

Element UI 的上传控件勾了 `multiple` 之后，看起来是一次选中多张图，实际仍是每个文件各打一条请求。图片一多，接口会被同时打满，后端若按「同一时刻只收一个文件」来写，就会出现部分成功、部分覆盖的情况。官方议题里有人提过希望合并成一次提交，维护者回复该能力不在计划内，需要自己收口。

## 解决方案

思路是关掉自动上传，等用户点提交时，从组件实例上取出已选文件，自己组一个 `FormData`，用一次 `XMLHttpRequest` 送出。

模板只负责选文件和触发提交：

```html
<el-upload
  class="batch-uploader"
  ref="albumUploader"
  name="albumParts"
  :headers="authHeaders"
  :action="albumEndpoint"
  :auto-upload="false"
  multiple
>
  <el-button slot="trigger" size="small" type="primary">选择图片</el-button>
  <el-button style="margin-left: 8px;" size="small" type="success" @click="pushAlbum">开始上传</el-button>
</el-upload>
```

提交函数从 ref 上拿 `uploadFiles`、`action` 和附加字段。成功后清空队列，失败只记状态码，避免把半截响应当成成功。

```js
pushAlbum() {
  const { uploadFiles, action, data } = this.$refs.albumUploader
  this.postAlbum({
    uploadFiles,
    data,
    action,
    success: () => {
      this.$refs.albumUploader.clearFiles()
    },
    error: (status) => {
      console.warn('上传未完成', status)
    }
  })
}
```

真正发请求的函数把每个原始文件追加到同一个字段名上，附加参数按键值再追加。请求头里的凭证单独设置，不要指望 `:headers` 在自定义请求里自动生效——那个属性只对组件内置上传有效。

```js
postAlbum({ uploadFiles, data, action, success, error }) {
  const form = new FormData()
  uploadFiles.forEach((item) => form.append('albumParts', item.raw))
  Object.keys(data || {}).forEach((key) => form.append(key, data[key]))

  const xhr = new XMLHttpRequest()
  xhr.open('POST', action, true)
  xhr.setRequestHeader('X-Access-Ticket', sessionStorage.getItem('accessTicket') || '')
  xhr.onreadystatechange = function () {
    if (xhr.readyState !== 4) return
    if (xhr.status >= 200 && xhr.status < 300) success && success(xhr.responseText)
    else error && error(xhr.status)
  }
  xhr.send(form)
}
```

后端用列表接收同名文件：

```java
@PostMapping("/albums")
public Object saveAlbum(List<MultipartFile> albumParts) {
  // 逐个落盘并返回每张图的访问地址
  return albumParts.size();
}
```

抓包时应看到一条请求里有多段文件，响应里是一批结果，而不是 N 条几乎同时发出的单文件请求。若 `item.raw` 为空，说明文件还停在组件的待上传队列之外，检查是不是误开了 `auto-upload`。
