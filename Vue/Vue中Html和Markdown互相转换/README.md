# Vue中Html和Markdown互相转换

## 前言

把站内文章导出成 Markdown，或把编辑器里的 Markdown 回写成 HTML，可以放在浏览器里做，也可以放在服务端做。浏览器做不占机器算力，适合预览和导出；服务端做适合批量任务和统一排版。下面各给一条可替换的实现，避免和某一篇教程的示例字符串绑死。

## 前台处理

### MarkdownToHtml

#### 安装

浏览器侧把 Markdown 转成 HTML，可以用 `marked`。它没有 Vue 运行时依赖，打包体积也小。

```bash
npm install marked --save
```

也可以用 CDN：

```html
<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
```

#### Markdown 转成 Html

```js
import { marked } from 'marked'

const source = '## 实验记录\n\n- 样品 A\n- 样品 B'
const html = marked.parse(source)
```

默认会把原文里的 HTML 原样留下。若文章来自访客，先过一遍消毒再 `v-html`，见同目录里关于 XSS 的笔记。

### HtmlToMarkdown

#### 安装

反向转换用 `turndown`。它按规则把常见块级标签收成 Markdown 语法。

```bash
npm install turndown --save
```

CDN：

```html
<script src="https://unpkg.com/turndown/dist/turndown.js"></script>
```

#### 使用

```js
import TurndownService from 'turndown'

const service = new TurndownService({ headingStyle: 'atx', bulletListMarker: '-' })
const markdown = service.turndown('<h2>实验记录</h2><ul><li>样品 A</li></ul>')
```

复杂表格和自定义属性默认会被丢掉。导出前在页面上先提示「仅保留标题、列表、链接和图片」。

## 后端处理

服务端若用 Java，可以用 `commonmark-java` 做正向转换，再用 jsoup 把 HTML 收成纯文本后按自己的规则拼 Markdown。下面只演示正向，反向用 jsoup 取文本，避免引入过重的全家桶。

### 引入依赖

```xml
<dependency>
  <groupId>org.commonmark</groupId>
  <artifactId>commonmark</artifactId>
  <version>${commonmark.version}</version>
</dependency>
```

### MarkdownToHtml

```java
public static String markdownToHtml(String markdown) {
  Parser parser = Parser.builder().build();
  HtmlRenderer renderer = HtmlRenderer.builder().escapeHtml(true).build();
  return renderer.render(parser.parse(markdown));
}
```

`escapeHtml(true)` 会把原文里的裸标签转义，适合评论；编辑器正文若允许内嵌 HTML，再改成 `false`。

### HtmlToMarkdown

```java
public static String htmlToMarkdown(String html) {
  Document doc = Jsoup.parse(html);
  StringBuilder out = new StringBuilder();
  for (Element h : doc.select("h1, h2, h3, p, li")) {
    String tag = h.tagName();
    if (tag.equals("h1")) out.append("# ");
    else if (tag.equals("h2")) out.append("## ");
    else if (tag.equals("h3")) out.append("### ");
    else if (tag.equals("li")) out.append("- ");
    out.append(h.text()).append('\n');
  }
  return out.toString();
}
```

这只覆盖标题、段落和列表。图片要单独遍历 `img`，写成 `![](src)`。不要把整段 HTML 当纯文本导出，否则读者会看到成堆标签。
