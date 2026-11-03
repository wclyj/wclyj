按宽度反推高度，让封面在不同列宽下保持同一比例。

1、安装指令，版本按锁文件里的来：

```bash
npm install vue-proportion-directive --save
```

2、在入口注册。指令会按元素当前宽度乘上给定比值写回高度。

```js
import proportion from 'vue-proportion-directive'
Vue.use(proportion)
```

3、比值是高度除以宽度。0.75 表示 4:3 的横图，1.2 表示偏高的竖图。

```html
<div class="card" style="width: 25%; float: left" v-proportion="0.75"></div>
<div class="card" style="width: 25%; float: left" v-proportion="1.2"></div>
<div class="card" style="width: 40%; float: left" v-proportion="0.56"></div>
```

坑：指令只在绑定时算一次。列宽会随窗口变化时，要在 `resize` 里重新触发，否则高度停在第一次的像素。图片本身再用 `object-fit: cover`，避免比例框里出现拉伸。
