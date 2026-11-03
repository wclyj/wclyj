# Vue中使用Vue-cropper进行图片裁剪

## 前言

封面如果原样上传，列表里经常被压扁或留出大块空白。上传前按固定比例裁一刀，前台展示就稳定。头像可以用圆形裁剪，横幅更适合按矩形比例裁。

## 图片裁剪

常见两套组件：

- [vue-cropper](https://github.com/xyxiao001/vue-cropper)：裁剪框比例、输出格式都能配，适合封面和横幅。
- [vue-image-crop-upload](https://github.com/dai-siki/vue-image-crop-upload)：交互更像头像上传，框是固定的，矩形比例不好调。

封面走第一套，头像可以留第二套。裁完得到的是 base64 或 blob，再交给原来的上传接口，不要在裁剪组件里另写一套存储。

## 引入Vue-cropper

```bash
npm install vue-cropper --save
```

按需引入，不必 `Vue.use` 全局注册：

```js
import { VueCropper } from 'vue-cropper'
```

```js
components: { VueCropper }
```

## 完整代码

下面把裁剪框锁成 16:9，输出 jpeg。选图后先做本地预览，点「完成」再取出裁剪结果。

```html
<template>
  <div class="crop-page">
    <div class="stage">
      <vue-cropper
        ref="bannerCrop"
        :img="draft.url"
        :output-size="draft.quality"
        :output-type="draft.mime"
        :info="true"
        :can-scale="true"
        :auto-crop="true"
        :auto-crop-width="480"
        :auto-crop-height="270"
        :fixed="true"
        :fixed-number="[16, 9]"
        :enlarge="2"
      />
    </div>
    <label class="pick" for="bannerFile">选择图片</label>
    <input
      id="bannerFile"
      class="file"
      type="file"
      accept="image/png, image/jpeg, image/webp"
      @change="onPick"
    >
    <button type="button" class="pick" @click="onCut">完成裁剪</button>
    <img v-if="resultUrl" class="preview" :src="resultUrl" alt="裁剪结果">
  </div>
</template>

<script>
import { VueCropper } from 'vue-cropper'

export default {
  components: { VueCropper },
  data() {
    return {
      resultUrl: '',
      draft: {
        url: '',
        quality: 0.9,
        mime: 'jpeg'
      }
    }
  },
  methods: {
    onPick(event) {
      const file = event.target.files[0]
      if (!file) return
      if (!/^image\/(png|jpeg|webp)$/.test(file.type)) {
        this.$message.warning('只接收 png、jpeg、webp')
        return
      }
      const reader = new FileReader()
      reader.onload = () => {
        const buf = reader.result
        this.draft.url = typeof buf === 'string'
          ? buf
          : URL.createObjectURL(new Blob([buf]))
      }
      reader.readAsArrayBuffer(file)
    },
    onCut() {
      this.$refs.bannerCrop.getCropData((dataUrl) => {
        this.resultUrl = dataUrl
        this.uploadBlob(this.dataUrlToBlob(dataUrl))
      })
    },
    dataUrlToBlob(dataUrl) {
      const [head, body] = dataUrl.split(',')
      const mime = head.match(/:(.*?);/)[1]
      const binary = atob(body)
      const bytes = new Uint8Array(binary.length)
      for (let i = 0; i < binary.length; i += 1) {
        bytes[i] = binary.charCodeAt(i)
      }
      return new Blob([bytes], { type: mime })
    },
    uploadBlob(blob) {
      const form = new FormData()
      form.append('banner', blob, 'banner.jpg')
      // 交给业务上传接口，返回访问地址后再写入表单
      return form
    }
  }
}
</script>

<style>
.crop-page { max-width: 640px; margin: 24px auto; }
.stage { height: 320px; background: #111; }
.file { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); }
.pick {
  display: inline-block;
  margin: 12px 8px 0 0;
  padding: 8px 14px;
  border: 0;
  border-radius: 4px;
  background: #2f6fed;
  color: #fff;
  cursor: pointer;
}
.preview { display: block; margin-top: 16px; max-width: 100%; }
@media (max-width: 720px) {
  .stage { height: 240px; }
}
</style>
```

`autoCropWidth` 只有在 `autoCrop` 打开时才生效。`fixedNumber` 决定拖动时的宽高比，和自动框的初始像素不是同一件事。`enlarge` 用来提高导出分辨率，设得过大时 base64 会很长，接口要看请求体限制。

读文件用 `readAsArrayBuffer` 再包成 blob URL，比直接 `readAsDataURL` 省一次大字符串。组件销毁时若还持有 `URL.createObjectURL` 的结果，记得 `URL.revokeObjectURL`，否则反复选图会把内存顶上去。

常用开关不要和自动框的初始像素混在一起看：

| 字段 | 作用 | 建议 |
| --- | --- | --- |
| fixed | 拖动时是否锁比例 | 封面打开，自由裁剪关掉 |
| fixedNumber | 宽高比，例如 `[16, 9]` | 和设计稿一致 |
| autoCrop | 打开后立刻出现裁剪框 | 需要默认框时打开 |
| autoCropWidth / Height | 框的初始像素 | 只在 autoCrop 为真时生效 |
| outputSize | 压缩质量，0 到 1 | 封面 0.9 足够，头像可以再低 |
| outputType | 导出格式 | 照片用 jpeg，需要透明再用 png |
| enlarge | 导出相对框的放大倍数 | 2 左右，再大请求体会超限 |
| canScale | 是否允许滚轮缩放原图 | 原图很大时打开 |

头像和封面不要共用一个实例。头像把 `fixedNumber` 设成 `[1, 1]`，并用 `getCropBlob` 直接拿二进制，少一次 base64 膨胀。封面继续用 `getCropData` 只是为了在对话框里预览，真正上传仍走 `dataUrlToBlob`。

```js
onAvatarCut() {
  this.$refs.avatarCrop.getCropBlob((blob) => {
    const form = new FormData()
    form.append('portrait', blob, 'portrait.jpg')
    this.$emit('picked', form)
  })
}
```

选图按钮若用 `<label for>`，点一次会触发隐藏的 file input。第二次选同一张图时，`change` 不会再次触发，因为值没变。在 `onPick` 末尾把 `event.target.value = ''`，才能重复选同一文件。

裁剪框超出原图时，导出的边缘会是空白。把 `centerBox` 设为真，框会被限制在图片内部。移动端高度不要写死 320 像素以上，否则工具栏被顶出首屏，用户找不到「完成」。

销毁前释放地址：

```js
beforeDestroy() {
  if (this.draft.url && this.draft.url.startsWith('blob:')) {
    URL.revokeObjectURL(this.draft.url)
  }
}
```

接口若限制 1MB，先看 `outputSize` 而不是先缩 `enlarge`。质量从 0.9 降到 0.7，体积通常能掉一截，清晰度仍够列表页使用。不要在裁剪完成前就把原图上传，否则存储里会留下未裁的大图，列表仍显示变形的那一张。
