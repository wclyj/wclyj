# 在 Vue 工程里接入 iconfont

## 前言

小图标换成矢量后，放大不会糊，也方便跟着主题改颜色。可选来源主要有两处：

- [iconfont](https://www.iconfont.cn/)
- [Font Awesome](https://fontawesome.com/)

Font Awesome 的免费集可以用于商业项目。iconfont 上很多图形是作者上传的，商用前要看清该图标是否允许，个人学习和内部系统一般没有问题。

## 采购矢量图标

打开 iconfont，用关键词搜到需要的图形，点「添加入库」。选完后打开右上角购物车，选择「添加至项目」。没有项目就先建一个，名字和前端仓库对应，方便以后增补。

项目页里选「下载至本地」，得到一个压缩包。不要只截图保存 Unicode，之后增删图标时 Unicode 可能变。

## 引入矢量图标

解压后除了预览页，还会有样式、字体文件和一份图标对照页。对照页里每个图标旁边有一串 `&#x....;`。

把字体和 `iconfont.css` 拷到 `src/assets/iconfont/`，预览页不必进仓库。在入口引入样式：

```js
import '@/assets/iconfont/glyph.css'
```

若官方包仍叫 `iconfont.css`，导入路径跟文件名走。页面里用样式类加上对应编码：

```html
<span class="iconfont">&#xe612;</span>
<span class="iconfont">&#xe7a4;</span>
```

类名必须和样式表里定义的一致。只写编码、不写类名时，浏览器会把它当成普通文字。颜色用 `color` 改，大小用 `font-size` 改。更新图标后要连同字体文件一起替换，只改样式而沿用旧的字体文件，新编码会显示成方框。多色图标不要用字体方式，字体只能单色，需要多色时改用项目提供的 SVG 下载。
