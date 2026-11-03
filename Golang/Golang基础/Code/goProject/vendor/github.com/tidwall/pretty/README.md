# Pretty：把 JSON 排版或压紧

`pretty` 提供一组速度很快的 JSON 排版函数：给人读时展开缩进，要省流量时去掉空白。它不负责解析成结构体，只处理字节层面的格式。

## 安装

本机已有 Go 时执行：

```sh
go get github.com/tidwall/pretty
```

模块会进入当前项目的依赖列表。

## Pretty

原始文本挤在少数几行里：

```json
{"label":{"given":"Lin","family":"Zhao"},"years":29,
"pets":["Mochi","Nori"],
"fav.song":"River","mates":[{"given":"Kai","family":"Chen","years":31}]}
```

调用：

```go
view := pretty.Pretty(raw)
```

得到带缩进的对象：键值分行，数组在不超过宽度时仍可以留在一行，嵌套对象各自缩进。

## Color

`Color` 给已经排好的 JSON 加上终端颜色，方便直接打到屏幕上。颜色写在 ANSI 转义里，重定向到文件时这些控制符也会被保存，所以只在交互终端上使用。先 `Pretty` 再 `Color`，顺序不要反：压紧后的单行文本也能上色，但可读性主要来自分行。

```go
colored := pretty.Color(view, nil)
```

第二个参数是样式。传入 `nil` 时使用库自带的 `TerminalStyle`。要换颜色，就传入自己的样式结构，而不是改 JSON 原文。

## Ugly

与排版相反，`Ugly` 删掉多余空白，得到更短的载荷：

```go
compact := pretty.Ugly(raw)
```

结果是单行（或几乎没有装饰空格）的 JSON，语义与输入相同。适合写日志前先缩小体积。它不会重新排序键，也不会改数字的写法。输入本身不合法时，不要指望它顺便修好语法。

## 定制输出

`PrettyOptions` 接受选项，而不是写死缩进：

```go
type Options struct {
    Width    int    // 单行数组允许的最大列宽，默认 80
    Prefix   string // 每一行前面的前缀，默认空串
    Indent   string // 嵌套一层时追加的缩进，默认两个空格
    SortKeys bool   // 是否按键名排序，默认否
}
```

例如把缩进改成四个空格，并按字母序输出键：

```go
opt := &pretty.Options{Indent: "    ", SortKeys: true, Width: 60}
view = pretty.PrettyOptions(raw, opt)
```

`@pretty` 一类调用方还会用到 `prefix` 与 `width`。`prefix` 适合在每行前加注释符号或日志标记。若前缀本身含有空格，每一层缩进仍追加在前缀之后，不会把前缀算进 `Width`。`Width` 只限制单行数组，对象始终按 `Indent` 换行。

## 性能

作者曾在 15 英寸 MacBook Pro（2.8 GHz Intel Core i7）上用 Go 1.7 对比标准库 `encoding/json` 的 `Indent` / `Compact`。公开结果大致是：

| 项目 | 每次操作量级 | 分配 |
| --- | --- | --- |
| Pretty | 约 1.3 微秒 | 2 次分配 |
| Ugly | 约 0.4 微秒 | 1 次分配 |
| Ugly 原地 | 约 0.3 微秒 | 0 次分配 |
| json.Indent | 约 4.6 微秒 | 4 次分配 |
| json.Compact | 约 2.5 微秒 | 4 次分配 |

这些数字只说明当年那台机器上的相对差距：排版和压缩都比先解码再编码更快，因为路径上少了结构体往返。换硬件和 Go 版本后应重新压测，不要把表中的纳秒当成承诺。比较时用同一份 JSON，并关掉调试日志，否则分配次数会被测试框架本身干扰。

## 联系

作者 Josh Baker，常用 id 为 tidwall。

## 许可

源码按 MIT 发布，条款见仓库中的许可证文件。
