# GJSON 路径语法

GJSON 路径是一小段文本，用来从一份 JSON 里直接取出值，不必先把整份文档解码进结构体。

下面按组件说明这条路径怎么写。权威实现是模块 `github.com/tidwall/gjson`，在线试验场可以临时验证一条路径，地址不在本文展开。

- 路径由什么组成
- 基本取法
- 通配
- 转义
- 数组
- 条件查询
- 点号和竖线
- 修饰符
- 多路径

## 路径结构

一段路径由若干组件拼成，组件之间通常用 `.`。

除了点号，这些字符也有专门含义：`|`、`#`、`@`、`\`、`*`、`?`。读路径时从左到右，先处理当前组件，再看下一个分隔符。空格一般不属于路径的一部分，写在 Go 字符串里时不要为了对齐去加空格。

## 示例

下文都针对这份 JSON：

```json
{
  "user": {"given": "Lin", "family": "Zhao"},
  "years": 29,
  "pets": ["Mochi", "Nori", "Bean"],
  "fav.song": "River Flow",
  "mates": [
    {"given": "Kai", "family": "Chen", "years": 31, "apps": ["wx", "wb", "dy"]},
    {"given": "Mina", "family": "Zhou", "years": 52, "apps": ["wb", "dy"]},
    {"given": "Yue", "family": "Chen", "years": 36, "apps": ["wx", "dy"]}
  ]
}
```

### Basic

大多数时候只是按对象的键或数组下标取值。

```text
user.family            "Zhao"
user.given             "Lin"
years                  29
pets                   ["Mochi","Nori","Bean"]
pets.0                 "Mochi"
pets.1                 "Nori"
mates.1                {"given":"Mina","family":"Zhou","years":52}
mates.1.given          "Mina"
```

下标从 0 开始，写在点号后面，看起来和字段一样。

### Wildcards

键名里可以出现 `*` 和 `?`。`*` 匹配任意长度（含空串），`?` 只匹配一个字符。

```text
pet*.2                 "Bean"
p?ts.0                 "Mochi"
```

`pet*` 能对上 `pets`，再取下标 2。`p?ts` 用一个问号填上 `e`。

### Escape character

键本身含有 `.`、`*`、`?` 时，用反斜杠转义，否则点号会被当成路径分隔符。

```text
fav\.song              "River Flow"
```

写在 Go 的双引号字符串里时，反斜杠自己也要再转义一次；用原生字符串字面量则不必：

```go
res := gjson.Get(payload, "fav\\.song")
res = gjson.Get(payload, `fav\.song`)
```

### Arrays

`#` 专门处理数组。单独一个 `#` 表示长度。

```text
mates.#                3
mates.#.years          [31,52,36]
```

`#` 后面再跟字段，表示对每个元素取该字段，结果仍是数组。

### Queries

`#(...)` 取第一条满足条件的元素，`#(...)#` 取全部满足条件的元素。

比较运算有 `==`、`!=`、`<`、`<=`、`>`、`>=`。`%` 是简单的模式匹配（类似包含通配），`!%` 表示不匹配该模式。

```text
mates.#(family=="Chen").given      "Kai"
mates.#(family=="Chen")#.given     ["Kai","Yue"]
mates.#(years>40)#.family          ["Zhou"]
mates.#(given%"K*").family         "Chen"
mates.#(given!%"K*").family        "Zhou"
```

数组元素不是对象、只是字符串时，比较符右边可以不写字段名，直接对元素本身做判断：

```text
pets.#(!%"*o*")                    "Bean"
pets.#(%"*o*")#                    ["Mochi","Nori"]
```

查询可以套查询。先在 `apps` 里找等于 `wb` 的项，再收集这些朋友的名字：

```text
mates.#(apps.#(=="wb"))#.given     ["Kai","Mina"]
```

1.3.0 之前，查询括号写成 `#[...]`。1.3.0 把这种写法让给了多路径，查询改成圆括号。旧括号在下一个大版本之前仍可用，新代码不要再写。

### Dot vs Pipe

`.` 是默认分隔符，多数位置也可以换成 `|`，结果相同。差别出现在 `#` 之后：点号是在收集结果之前对每个元素继续走路径；竖线是先得到上一步的结果，再对这个整体取值。

```text
mates.0.given                         "Kai"
mates|0.given                         "Kai"
mates.0|given                         "Kai"
mates|0|given                         "Kai"
mates|#                               3
mates.#                               3
mates.#(family="Chen")#               [Kai 的对象, Yue 的对象]
mates.#(family="Chen")#.given         ["Kai","Yue"]
mates.#(family="Chen")#|given         不存在
mates.#(family="Chen")#.0             []
mates.#(family="Chen")#|0             Kai 的整条对象
mates.#(family="Chen")#.#             []
mates.#(family="Chen")#|#             2
```

拆开看：`mates.#(family="Chen")#` 先得到一个数组，里面是两条对象。

`.given` 会在返回前对每条对象取 `given`，所以得到名字数组。

`|given` 是在数组整体上找键 `given`。数组不是对象，这个键不存在。

`|0` 则把上一步的数组当成数组，取出下标 0，因此得到 Kai 那一条对象。`|#` 取的是这个结果数组的长度，得到 2。

### Modifiers

修饰符是路径里的一段自定义处理，以 `@` 开头。

例如 `@reverse` 把数组或对象成员倒序：

```text
pets.@reverse                ["Bean","Nori","Mochi"]
pets.@reverse.0              "Bean"
```

当前内建修饰符：

- `@reverse`：反转数组，或反转对象成员顺序
- `@ugly`：去掉 JSON 中的空白
- `@pretty`：排成便于阅读的缩进
- `@this`：当前元素；用在根上可以取回整份文档
- `@valid`：检查这段 JSON 是否合法
- `@flatten`：把嵌套数组合并成一层
- `@join`：把多个对象合并成一个对象

#### Modifier arguments

修饰符可以带一个可选参数，参数既可以是合法 JSON，也可以是普通字符。

`@pretty` 接受一个 JSON 对象。下面要求按键名排序：

```text
@pretty:{"sortKeys":true}
```

排版后的键会按字母序出现，嵌套对象同样处理。`@pretty` 还认识 `indent`、`prefix`、`width`，分别控制缩进、行前缀和单行数组的列宽。细节与 pretty 库的选项一致。

#### Custom modifiers

也可以登记自己的修饰符。下面按参数把 JSON 文本转成大写或小写：

```go
gjson.AddModifier("fold", func(body, arg string) string {
    switch arg {
    case "up":
        return strings.ToUpper(body)
    case "down":
        return strings.ToLower(body)
    default:
        return body
    }
})
```

```text
pets.@fold:up                  ["MOCHI","NORI","BEAN"]
pets.@fold:down.@reverse       ["bean","nori","mochi"]
```

修饰符收到的是 JSON 文本，不是已经解码的 Go 值。返回值也应是一段 JSON 文本，后续路径会继续在这段文本上解析。

### Multipaths

从 1.3.0 起，可以把多条路径合成一份新的 JSON。花括号 `{...}` 得到对象，方括号 `[...]` 得到数组，里面用逗号分隔各条路径。

```text
{user.given,years,"chen":mates.#(family="Chen")#.given}
```

这里取出名字、年龄，以及姓 Chen 的朋友的名。可以写可选的键，例如 `"chen"`，用来指定结果里的字段名。不写时，就用路径末端的字段名；本例里 `user.given` 的末端是 `given`。若连字段名也推不出来，就用 `_`。

结果类似：

```json
{"given":"Lin","years":29,"chen":["Kai","Yue"]}
```

方括号形式则把各条路径的值按顺序放进数组，不再保留键名。多路径适合一次性拼出接口需要的摘要，避免在 Go 里反复调用 `Get`。

路径不存在时，`Get` 得到的 `Result` 用 `Exists()` 判断为假，`String()` 往往是空串，不会因为缺键而 panic。数字可以用 `Int()`、`Float()`，布尔用 `Bool()`。要遍历 `mates.#.given` 这种数组结果，调用 `Array()` 得到 `[]Result`。这些方法只读取，不改原文。

写路径时先用点号把对象走通，确认下标和 `#` 的位置，再加查询。查询条件里的字符串要加引号；数字不用加。把整段路径放进 Go 的原生字符串字面量，反斜杠只需要写一遍。
