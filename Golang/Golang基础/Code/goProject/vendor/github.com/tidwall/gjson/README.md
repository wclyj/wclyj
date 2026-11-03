# gjson

gjson 用一行调用从 JSON 文本里取出某个值。路径用点号分隔，也能遍历对象和数组，还能把多行 JSON 当成一组记录来读。它不做完整的结构体反序列化，适合只关心少数字段、又不想定义类型的场景。

要改 JSON 可以用同作者的 sjson，命令行场景另有专用工具。本篇只讲读取。

## Installing

安装 Go 之后，在模块目录执行：

```bash
go get github.com/tidwall/gjson
```

模块会把版本写进 `go.mod`。业务代码里 `import "github.com/tidwall/gjson"` 即可。

## Get a value

`Get` 的第一个参数是 JSON 文本，第二个是路径。找到就立刻返回一个 `Result`，不必先变成 `map`。

```go
package main

import "github.com/tidwall/gjson"

const doc = `{"user":{"given":"Ann","family":"Cole"},"years":31}`

func main() {
    println(gjson.Get(doc, "user.family").String())
}
```

输出 `Cole`。一次要取多个路径用 `GetMany`，输入本身是 `[]byte` 时用 `GetBytes`，避免先把字节转成字符串再复制一份。

## Path Syntax

路径是用点号连起来的键。键里可以出现 `*` 和 `?` 做简单通配。数组用数字下标。`#` 表示元素个数，也能在取子路径时展开每一项。点号和通配符若是键名的一部分，用反斜杠转义。

示例文档：

```json
{
  "user": {"given": "Mia", "family": "Ng"},
  "years": 29,
  "pets": ["bird", "cat", "dog"],
  "fav.city": "Ningbo",
  "mates": [
    {"given": "Bo", "family": "Ng", "years": 40, "nets": ["ig", "x"]},
    {"given": "Cy", "family": "Ou", "years": 55, "nets": ["x"]},
    {"given": "Di", "family": "Ng", "years": 33, "nets": ["ig"]}
  ]
}
```

对应结果：

```text
user.family            -> Ng
years                  -> 29
pets                   -> ["bird","cat","dog"]
pets.#                 -> 3
pets.1                 -> cat
pet*.2                 -> dog
p?ts.0                 -> bird
fav\.city              -> Ningbo
mates.#.given          -> ["Bo","Cy","Di"]
mates.1.family         -> Ou
```

数组还可以写查询。`#(...)` 取第一条匹配，`#(...)#` 取全部匹配。比较运算包括 `==`、`!=`、`<`、`<=`、`>`、`>=`。`%` 是简单的相似匹配，`!%` 表示不相似，星号在模式里代表一段任意字符。

```text
mates.#(family=="Ng").given     -> Bo
mates.#(family=="Ng")#.given    -> ["Bo","Di"]
mates.#(years>35)#.family       -> ["Ng","Ou"]
mates.#(given%"B*").family      -> Ng
mates.#(given!%"B*").family     -> Ou
mates.#(nets.#(=="x"))#.given   -> ["Bo","Cy"]
```

1.3.0 之前查询写成 `#[...]`。为了和后来的多路径语法区分，括号改成了圆括号。旧写法在下一个主版本之前仍被接受，新代码不要再用方括号查询。

## Result Type

JSON 的字符串、数字、布尔和 null 会落进 `Result` 的对应字段。数组和对象则保留原始 JSON 片段，需要时再往下取。

`Result` 里能直接看到的内容：

```text
bool     JSON 布尔
float64  JSON 数字
string   JSON 字符串
nil      JSON null
```

常用字段：

```go
result.Type    // String、Number、True、False、Null 或 JSON
result.Str
result.Num
result.Raw     // 原始片段
result.Index   // 该片段在原文中的字节下标，0 表示未知
```

常用方法：

```go
result.Exists() bool
result.Value() interface{}
result.Int() int64
result.Uint() uint64
result.Float() float64
result.String() string
result.Bool() bool
result.Time() time.Time
result.Array() []gjson.Result
result.Map() map[string]gjson.Result
result.Get(path string) gjson.Result
result.ForEach(func(key, value gjson.Result) bool)
result.Less(token gjson.Result, caseSensitive bool) bool
```

`Value()` 返回空接口，具体类型要断言：

```text
布尔   -> bool
数字   -> float64
字符串 -> string
null   -> nil
数组   -> []interface{}
对象   -> map[string]interface{}
```

`Array()` 在当前结果不是数组时，返回只含自身的一片；路径不存在时返回空切片。因此要先 `Exists`，再决定是否把单个值当成数组用。

### 64-bit integers

JSON 数字默认进 `float64`，超过 2^53 的整数会丢精度。`Int` 和 `Uint` 改从原始文本读，能覆盖全部 64 位：

```go
result.Int()  // 有符号，约 -9.22e18 到 9.22e18
result.Uint() // 无符号，0 到约 1.84e19
```

订单号、雪花 ID 这类字段用这两个方法，不要用 `Float` 再转整数。

## Modifiers and path chaining

从 1.2 起，路径里可以写修饰器，并用 `|` 把多段路径串起来。修饰器对当前 JSON 做一次变换，后面的路径再在变换结果上取值。

例如先把数组倒序，再取第一项：

```text
pets|@reverse        -> ["dog","cat","bird"]
pets|@reverse|0      -> dog
```

内置修饰器：

- `@reverse`：倒序数组，或倒序对象的成员
- `@ugly`：去掉空白，压成一行
- `@pretty`：加上缩进，便于阅读
- `@this`：当前元素本身，也可用来取整篇文档
- `@valid`：检查这段 JSON 是否合法
- `@flatten`：把嵌套数组摊平一层
- `@join`：把多个对象合并成一个对象

### Modifier arguments

修饰器可以带参数，参数既可以是一段 JSON，也可以是普通字符。`@pretty` 接受一个对象，用来控制排版：

```text
@pretty:{"sortKeys":true}
```

键会按字母序输出，嵌套对象同样受影响。可选键还有 `indent`、`prefix` 和 `width`，分别控制缩进、每行前缀和换行宽度。不需要这些选项时，单独写 `@pretty` 就使用默认缩进。

### Custom modifiers

`AddModifier` 注册自己的变换。函数收到当前 JSON 文本和参数字符串，返回新的 JSON 文本。下面按参数把文本改成大写或小写：

```go
gjson.AddModifier("case", func(json, arg string) string {
    switch arg {
    case "upper":
        return strings.ToUpper(json)
    case "lower":
        return strings.ToLower(json)
    default:
        return json
    }
})
```

```text
pets|@case:upper          -> ["BIRD","CAT","DOG"]
pets|@case:lower|@reverse -> ["dog","cat","bird"]
```

修饰器是进程级注册，名字不要和内置的 `@reverse` 等冲突。它收到的是文本，不是 `Result`，返回值必须仍是合法 JSON，后面的路径才能继续解析。

## JSON Lines

多行文档里每一行是一个独立 JSON 对象时，路径以 `..` 开头，库会把各行看成数组元素。

```text
{"name":"Ann","years":61}
{"name":"Bo","years":34}
{"name":"Cy","years":57}
{"name":"Di","years":44}
```

```text
..#                  -> 4
..1                  -> {"name":"Bo","years":34}
..3                  -> {"name":"Di","years":44}
..#.name             -> ["Ann","Bo","Cy","Di"]
..#(name="Cy").years -> 57
```

`ForEachLine` 逐行回调。回调返回 false 时停止：

```go
gjson.ForEachLine(raw, func(line gjson.Result) bool {
    println(line.Get("name").String())
    return true
})
```

空行和非法行要事先过滤，这个前缀假定每一行自己是完整 JSON。

## Get nested array values

从对象数组里抽出同一个字段，在路径中写 `#.字段`：

```json
{
  "crew": [
    {"given": "Ann", "family": "Cole"},
    {"given": "Bo", "family": "Diaz"},
    {"given": "Cy", "family": "East"}
  ]
}
```

```go
got := gjson.Get(raw, "crew.#.family")
for _, name := range got.Array() {
    println(name.String())
}
```

会依次打印 Cole、Diaz、East。带条件时把查询写在 `#` 后面：

```go
one := gjson.Get(raw, `crew.#(family="Diaz").given`)
println(one.String()) // Bo
```

反引号字符串可以少写一层转义。条件两边的引号属于路径语法，不是 Go 的字符串结束符。

## Iterate through an object or array

`ForEach` 遍历对象或数组。对象会同时给出键和值，数组的键是下标。回调返回 false 就停止，返回 true 继续。

```go
gjson.Get(raw, "crew").ForEach(func(key, value gjson.Result) bool {
    println(value.Get("given").String())
    return true
})
```

不要在回调里修改原文字符串。`Result` 里的 `Raw` 可能指向原文的一段，原文被替换后这些片段就失效了。

## Simple Parse and Get

`Parse` 先得到整篇文档的 `Result`，再多次 `Get`，避免对同一段文本重复做最外层的定位。下面三种写法结果相同：

```go
gjson.Parse(raw).Get("user").Get("family")
gjson.Get(raw, "user").Get("family")
gjson.Get(raw, "user.family")
```

只取一个字段时，第三种最短。要在同一文档上取十几个字段时，先 `Parse` 一次再分别 `Get`，意图更清楚。

## Check for the existence of a value

路径写错或字段缺失时，`Get` 仍返回 `Result`，但 `Exists` 为 false。不要把空字符串当成“字段存在且为空”的唯一证据，数字 0 和空串都可能是合法值。

```go
got := gjson.Get(raw, "user.family")
if !got.Exists() {
    println("missing family")
} else {
    println(got.String())
}

if gjson.Get(raw, "user.family").Exists() {
    println("family present")
}
```

## Validate JSON

`Get` 和 `Parse` 假定文本基本完整。非法 JSON 通常不会 panic，但可能返回空结果或截断后的片段。数据来自外部时先校验：

```go
if !gjson.Valid(raw) {
    return errors.New("bad json")
}
family := gjson.Get(raw, "user.family")
```

校验通过只说明语法合法，不说明业务字段一定存在，字段仍要 `Exists`。

## Unmarshal to a map

偶尔需要通用 map 时，可以走 `Value` 再断言。这会分配内存，失去“只取一个字段、零分配”的好处，只在字段集合不固定时使用。

```go
m, ok := gjson.Parse(raw).Value().(map[string]interface{})
if !ok {
    // 顶层不是对象
}
```

顶层若是数组，断言目标应是 `[]interface{}`。

## Working with Bytes

JSON 已经在 `[]byte` 里时，用 `GetBytes`，不要 `Get(string(data), path)`，后者会多一次字符串复制。

```go
got := gjson.GetBytes(data, "user.family")
```

若还要把 `Raw` 交回给只接受字节的 API，又希望少分配，可以按 `Index` 从原文切一片：

```go
got := gjson.GetBytes(data, path)
var raw []byte
if got.Index > 0 {
    raw = data[got.Index : got.Index+len(got.Raw)]
} else {
    raw = []byte(got.Raw)
}
```

`Index` 为 0 时无法区分“真的在开头”和“位置未知”，这时退回复制 `Raw`。这是尽力避免分配，不是所有路径都能切片成功。修饰器改写过的结果往往不再指向原文，只能走复制。

## Get multiple values at once

`GetMany` 一次传入多个路径，返回的切片长度与路径个数相同，顺序一一对应。某个路径不存在时，对应元素是不存在的 `Result`，不会让整个调用失败。

```go
parts := gjson.GetMany(raw, "user.given", "user.family", "years")
```

适合从一条日志里同时抽出编号、级别和消息。路径数组可以来自配置，但不要把不可信的路径直接拼进查询表达式，查询语法本身能匹配任意字段。

## Performance

和完整反序列化相比，单字段读取的分配通常是 0：它只扫描到目标位置就返回。把整篇解析成 `map[string]interface{}` 会为每个键和值分配接口和字符串。标准库 `encoding/json`、代码生成的解析器以及其他流式库在“取出一个嵌套字段”上往往更慢，或至少分配更多。

一份用于对照的文档含有窗口、图片和文本三段嵌套。基准轮流读取下面三条路径：

```text
widget.window.name
widget.image.hOffset
widget.text.onMouseUp
```

公开的对照里，单次 `Get` 大约是数百纳秒、零分配；解码成 map 则是数微秒、数十次分配。具体数字随 CPU 和 Go 版本变化，结论只取相对关系：只读几个字段时用路径取值；要遍历并修改大部分字段时，仍应解码进结构体。

基准环境曾是一台 15 英寸的 MacBook Pro，处理器为 2.8 GHz 的 Core i7，Go 1.8。换机器后应自己跑基准，不要把旧数字写进性能承诺。

实际选择可以按数据形状来。日志、回调正文、只抽三五个字段的配置，用路径读取，省掉结构体定义。需要校验每个字段、或者几乎每个键都要进业务结构时，`encoding/json` 配结构体更清晰，编译期还能发现字段名写错。两者可以并存：先 `Valid`，再用 gjson 判断类型分支，最后只把命中的那一段 `Raw` 交给 `json.Unmarshal`。

多路径一次取出时，注意返回切片与参数对齐，不要用下标假设某个路径一定存在：

```go
doc := `{"id":"a18","level":"warn","msg":"disk"}`
got := gjson.GetMany(doc, "id", "level", "msg", "host")
for i, item := range got {
    if !item.Exists() {
        fmt.Println(i, "absent")
        continue
    }
    fmt.Println(i, item.String())
}
```

这里前三项有值，`host` 不存在，循环会打印 absent，而不会把空串误当成主机名。查询数组时也一样：`#(...)` 没有命中时 `Exists` 为 false，不要对它调用会假设类型的方法之后直接当业务主键用。

路径里的通配符只适合键名模式，不适合把整篇文档扫成业务规则引擎。复杂条件（跨字段计算、日期区间、单位换算）先取出原始字段，再在 Go 里判断。把规则写进路径，调试时很难知道是路径写错还是数据缺失。

## Contact

库由 Josh Baker 维护，讨论和问题通过代码仓库的 issue 进行。使用方式以当前版本的路径语法为准。

## License

源码使用 MIT 许可证。引入项目时保留版权声明，可以闭源调用，但不要把许可头从复制出的源文件里删掉。
