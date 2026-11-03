# HTTP 与模板

## 来源

要点来自公开的模板教程，原文地址不在这里重复。

## 介绍

输出 HTML 时用 `html/template`。它和 `text/template` 的调用方式一致，但会按上下文转义，避免把用户输入直接拼进页面造成注入。纯文本、邮件正文才考虑 `text/template`。

## 模板与渲染

服务端渲染仍会碰到：页面骨架固定，只有标题、列表之类的数据在变。模板就是事先写好的 HTML，渲染则是用数据替换里面的占位。

其他语言也有同类引擎，例如 Flask 常用的 Jinja2。Go 把这套能力放进了标准库，不需要再引一个第三方引擎才能完成替换。

## Go语言的模板引擎

标准库提供 `text/template` 与 `html/template`。使用时记住四条：

1. 文件后缀常见是 `.tmpl` 或 `.tpl`，也可以自定，但内容必须是 UTF-8。
2. 动作写在 `{{` 与 `}}` 之间。
3. `.` 表示当前数据。传入结构体或映射后，用 `{{.Field}}` 取字段或键。
4. 分界符以外的字节原样输出。

## 模板引擎的使用

整个过程分成三步：写模板、解析、执行。

### 定义模板文件

按下一节的语法把占位写进 HTML。解析之前，文件只是普通文本。

### 解析模板文件

常用入口：

- `Parse`：解析一段字符串，适合测试或把模板嵌在源码里
- `ParseFiles`：按路径读一个或多个文件
- `ParseGlob`：按通配符一次载入多份文件

也可以先 `template.New("页面名")` 得到空模板，再调用上面的方法。名字要和后面 `ExecuteTemplate` 使用的名字一致。

### 模板渲染

执行就是把数据填进去，结果写到 `io.Writer`。HTTP 里这个 Writer 通常就是 `http.ResponseWriter`。

- `Execute` 使用当前模板
- `ExecuteTemplate` 在一组已解析的模板里按名字挑一份

### 基本示例

#### 定义模板文件

`views/welcome.tmpl`：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head><title>welcome</title></head>
<body>
  <p>静态文字保持不变</p>
  <p>你好，{{.}}</p>
</body>
</html>
```

`{{.}}` 会被换成传入的整个值。这里传入的是字符串，所以页面上出现这串文字。

#### 解析和渲染模板文件

```go
package main

import (
    "html/template"
    "log"
    "net/http"
    "os"
    "path/filepath"
)

func welcome(w http.ResponseWriter, r *http.Request) {
    root, err := os.Getwd()
    if err != nil {
        http.Error(w, "cwd", http.StatusInternalServerError)
        return
    }
    path := filepath.Join(root, "views", "welcome.tmpl")
    page, err := template.ParseFiles(path)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if err = page.Execute(w, "江湾"); err != nil {
        log.Println(err)
    }
}

func main() {
    http.HandleFunc("/welcome", welcome)
    log.Fatal(http.ListenAndServe(":9090", nil))
}
```

路径用 `filepath.Join` 拼接，避免把 Windows 的反斜杠写死在代码里。浏览器打开 `http://127.0.0.1:9090/welcome`，第二段会显示“你好，江湾”。这是最小的一次渲染。

## 模板语法

### {{.}}

动作都在双花括号里。单独的点表示“当前数据”。把结构体传进去之后，点后面跟导出字段名。

```go
type Profile struct {
    Nick   string
    City   string
    Score  int
}

func profile(w http.ResponseWriter, r *http.Request) {
    page, err := template.ParseFiles("views/profile.tmpl")
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    item := Profile{Nick: "江湾", City: "杭州", Score: 12}
    _ = page.Execute(w, item)
}
```

对应模板：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head><meta charset="UTF-8"><title>profile</title></head>
<body>
  <p>昵称 {{.Nick}}</p>
  <p>城市 {{.City}}</p>
  <p>分数 {{.Score}}</p>
</body>
</html>
```

访问 `/profile` 时，三段分别换成结构体字段。字段必须导出，小写字段在模板里取不到。

映射用键名，写法相同：

```go
bag := map[string]any{
    "Nick":  "江湾",
    "Score": 18,
    "City":  "杭州",
}
_ = page.Execute(w, bag)
```

若一次要同时交出映射和结构体，再包一层：

```go
view := map[string]any{
    "bag":     bag,
    "profile": item,
}
_ = page.Execute(w, view)
```

模板里就写成 `{{.bag.Nick}}` 和 `{{.profile.City}}`。

### 注释

```template
{{/* 这里的说明不会出现在输出里，可以跨行。
     注释不能再套注释，星号要紧贴分界符。 */}}
```

### pipeline

能产出数据的表达式都叫 pipeline，`{{.}}` 和 `{{.Nick}}` 都是。竖线 `|` 把前一段的结果交给下一段的最后一个参数，方向和 shell 管道相同。

没有竖线时，单独一个取值表达式同样是 pipeline。竖线只是把多个命令串起来。

### 变量

动作里面可以用 `$` 开头的变量记住当前结果：

```template
{{$who := .Nick}}{{$who}}
```

后面的动作可以继续使用 `$who`，直到当前的 `if`、`range`、`with` 结束。

### 移除空格

模板里的换行和缩进会进到输出。在分界符旁加上减号可以吃掉相邻空白：`{{-` 去掉左侧空白，`-}}` 去掉右侧空白。

```template
{{- .Nick -}}
```

减号必须紧挨花括号，和表达式之间仍要留空格。

### 条件判断

```template
{{if pipeline}}分支甲{{end}}
{{if pipeline}}分支甲{{else}}分支乙{{end}}
{{if pipeline}}分支甲{{else if pipeline}}分支乙{{end}}
```

零值视为假：`false`、`0`、空串、空切片、空映射、nil 指针都会走到 else。

### range

`range` 遍历数组、切片、映射或通道。

```template
{{range pipeline}}每一项{{end}}
{{range pipeline}}每一项{{else}}没有元素时{{end}}
```

长度为 0 时，第一种什么都不输出；第二种执行 else。循环体内的 `.` 变成当前元素。映射遍历时可以写 `{{range $k, $v := .}}`。

### with

```template
{{with pipeline}}内层{{end}}
{{with pipeline}}内层{{else}}空数据{{end}}
```

pipeline 非空时，把 `.` 临时改成这个值再执行内层，结束后外面的点恢复原样。为空则第一种不输出，第二种走 else，并且不改点。

### 预定义函数

查找函数时先看模板自己的函数表，再看全局表。一般不在模板文件里定义函数，而是用 `Funcs` 注册。

全局函数里常用的有：

- `and`：返回第一个空参数，否则返回最后一个。`and x y` 类似“x 为真则取 y，否则取 x”。参数都会求值。
- `or`：返回第一个非空参数，否则返回最后一个。
- `not`：对单个参数取布尔否定。
- `len`：长度，结果是整数。
- `index`：多层下标。`index xs 1 2` 相当于 `xs[1][2]`，主体必须是数组、切片或映射。
- `print`、`printf`、`println`：对应 `fmt.Sprint`、`Sprintf`、`Sprintln`。
- `html`、`urlquery`、`js`：按各自上下文转义。在 `html/template` 里这几个被上下文相关的自动转义取代，不要再手动调用。
- `call`：把第一个参数当函数调用，其余是它的参数。该函数应返回一个值，或再加一个 `error`。error 非空时，整次执行失败。

### 比较函数

比较函数只接受基本类型（以及底层仍是基本类型的自定义类型，例如 `type Celsius float32`）。整数和浮点不能直接比。

```text
eq  相等
ne  不等
lt  小于
le  小于等于
gt  大于
ge  大于等于
```

只有 `eq` 可以接收两个以上的参数：第一个和后面每一个比较，全部相等才为真。

```template
{{if eq .City "杭州" "宁波"}}沿海{{end}}
```

### 自定义函数

自定义函数要么只返回一个值，要么第二个返回值是 `error`。注册必须发生在解析之前，因为解析时就要确认函数名存在。

```go
func greet(w http.ResponseWriter, r *http.Request) {
    praise := func(nick string) (string, error) {
        return nick + "，状态很好", nil
    }
    page := template.New("card.tmpl").Funcs(template.FuncMap{
        "praise": praise,
    })
    if _, err := page.ParseFiles("views/card.tmpl"); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    data := map[string]any{"Nick": "江湾", "Score": 18}
    _ = page.Execute(w, data)
}
```

模板里写 `{{praise .Nick}}`。`New` 的名字要和文件名（或 `define` 的名字）对上，否则执行时会找不到模板。

浏览器打开对应路由，昵称后面会接上函数拼接的那半句。

### 模板的嵌套template

一份模板可以用 `{{template "名字" pipeline}}` 拉入另一份。被拉入的模板既可以是单独文件，也可以是同一文件里的 `define`。

`page.tmpl`：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<body>
  <h1>片段组合</h1>
  {{template "menu.tmpl" .}}
  <hr>
  {{template "steps"}}
</body>
</html>

{{define "steps"}}
<ol>
  <li>热身</li>
  <li>练习</li>
  <li>拉伸</li>
</ol>
{{end}}
```

`menu.tmpl`：

```template
<ul>
  <li>目录</li>
  <li>记录</li>
  <li>检查</li>
</ul>
```

处理函数：

```go
func combo(w http.ResponseWriter, r *http.Request) {
    page, err := template.ParseFiles("views/page.tmpl", "views/menu.tmpl")
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    _ = page.Execute(w, Profile{Nick: "江湾", City: "杭州", Score: 18})
}
```

解析多个文件时，带 `define` 或被引用的文件要和主文件一起交给 `ParseFiles`。只解析主文件会导致执行期找不到 `menu.tmpl`。

### block

```template
{{block "content" .}}默认内容{{end}}
```

`block` 等价于先 `define` 再立刻 `template`。典型用途是做一套布局，子页面只替换其中一块。

`layout/base.tmpl`：

```template
<!DOCTYPE html>
<html lang="zh-CN">
<body>
  <main>
    {{block "content" .}}{{end}}
  </main>
</body>
</html>
```

`layout/home.tmpl` 重新定义同名块：

```template
{{template "base.tmpl" .}}
{{define "content"}}
  <p>首页正文</p>
{{end}}
```

```go
func home(w http.ResponseWriter, r *http.Request) {
    set, err := template.ParseGlob("layout/*.tmpl")
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    if err = set.ExecuteTemplate(w, "home.tmpl", nil); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

后解析到的同名 `define` 会覆盖先解析到的。不同业务都有 `home.tmpl` 时，有两种办法：

1. 在文件开头写 `{{define "唯一名字"}}`，执行时用这个名字。
2. 把文件分到子目录，用 `template.ParseGlob("layout/**/*.tmpl")` 一次装入。注意：标准库的通配实现并不总是支持 `**`，必要时改成明确的目录列表。

### 修改默认的标识符

Vue、Angular 也占用 `{{` `}}`。两边可以改一边。改 Go 这一侧用 `Delims`，而且要在解析之前调用。

```go
page, err := template.New("card.tmpl").
    Delims("{[", "]}").
    ParseFiles("views/card.tmpl")
```

模板改成：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<body>
  <p>昵称 {[.Nick]}</p>
  <p>城市 {[.City]}</p>
  <p>分数 {[.Score]}</p>
</body>
</html>
```

页面仍能显示字段。没有改到的 `{{` 会被当成普通文本或导致解析错误，整份文件的分界符要一起换。

## text/template与html/template的区别

`html/template` 在写入 HTML 时按位置转义，用来降低跨站脚本风险。同一段 `{{.}}` 出现在正文、属性或脚本里，转义规则不同。

模板若只是：

```template
<body>{{.}}</body>
```

传入 `` `<script>alert('demo')</script>` ``，页面上看到的是转义后的文本，脚本不会执行。

确定某段内容已经是安全的 HTML、必须原样输出时，可以注册一个函数，返回 `template.HTML`。这个类型是信任标记，标准库不再转义它。

```go
func raw(w http.ResponseWriter, r *http.Request) {
    page, err := template.New("raw.tmpl").Funcs(template.FuncMap{
        "trusted": func(s string) template.HTML { return template.HTML(s) },
    }).ParseFiles("views/raw.tmpl")
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    snippet := `<em>已审核的强调</em>`
    _ = page.Execute(w, snippet)
}
```

模板中写成 `{{. | trusted}}`。只对可信内容使用。用户输入仍应走默认转义。
