# match：极简通配

`match` 只做一件事：用两个通配符比较字符串。`*` 吃掉任意长度（可以是空），`?` 恰好对应一个字符。它不是正则引擎，没有分组和量词。

## 安装

```bash
go get github.com/tidwall/match
```

执行后模块会写入当前项目的 `go.mod`。

## 示例

```go
ok1 := match.Match("river", "*ver")
ok2 := match.Match("river", "?iver")
ok3 := match.Match("river", "r*r")
```

三条都为真：星号盖住前缀 `ri`，问号对上首字母 `r`，`r*r` 允许中间出现任意字符。

问号不能匹配空位，所以 `?iver` 对不上 `iver`。星号可以匹配空串，因此 `r*r` 也能对上只有两个 `r` 的情况。比较是大小写敏感的，`River` 和 `river` 不会视为相同。模式里若要表达字面量的星号或问号，这个库不提供转义，应改用别的匹配方式。

## 联系

作者 Josh Baker，常用 id 为 tidwall。

## 许可

源码按 MIT 发布，条款见仓库根目录的许可证文件。
