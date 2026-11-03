# Golang map详解

## map的介绍

map 是按键查找值的无序集合，在 Go 里属于引用类型。只声明不初始化时值是 `nil`，不能写入。

类型写法：

```go
map[KeyType]ValueType
```

- `KeyType`：键的类型，必须能比较相等，例如字符串、整数、布尔；切片不能当键
- `ValueType`：值的类型，可以是任意类型

零值是 `nil`。要用 `make` 分配哈希表，或用字面量直接写出键值。`make` 同样负责切片和 channel 的初始化。

先分配再逐个写入：

```go
account := make(map[string]string)
account["nick"] = "lin"
account["years"] = "28"
account["role"] = "dev"
fmt.Println(account)
fmt.Println(account["nick"])
```

声明时一次填入多对：

```go
account2 := map[string]string{
	"nick":  "chen",
	"years": "30",
	"role":  "ops",
}
fmt.Println(account2)
```

两种结果都是可用的 map。字面量即使写成空的 `map[string]string{}`，也已经分配，可以写入；`var m map[string]string` 则不行。

## 遍历map

用 `for range` 同时取出键和值。遍历次序不稳定，不能依赖某一次打印的先后。

```go
for key, value := range account2 {
	fmt.Printf("k=%s v=%s\n", key, value)
}
```

只要键时，值的位置写 `_`。

## 判断map中某个键值是否存在

用键读取时可以接两个结果：第一个是值，第二个是布尔值，表示键是否存在。键不存在时，值是值类型的零值，单看第一个结果无法区分“存了零值”和“根本没有这个键”。

```go
value, ok := account2["title"]
fmt.Println(value, ok)
```

`ok` 为 false 时不要把 `value` 当成有效业务数据。

## 使用delete()函数删除键值对

内建函数 `delete` 按键删掉一对。键不存在时什么也不做，不会 panic。

```text
delete(map对象, key)
```

- 第一个参数是要修改的 map
- 第二个参数是待删除的键

```go
delete(account2, "role")
fmt.Println(account2)
```

删除后再次读取 `"role"`，`ok` 为 false。

## 元素为map类型的切片

一组结构相似的记录可以放进切片，每个元素本身是一张 map。`make` 的长度要和准备放入的元素个数一致，否则按下标赋值会越界；也可以长度为 0，再用 `append`。

```go
roster := make([]map[string]string, 3)
roster[0] = map[string]string{"nick": "ann", "years": "19"}
roster[1] = map[string]string{"nick": "bo", "years": "21"}
roster[2] = map[string]string{"nick": "cyd", "years": "23"}
fmt.Println(roster)

for _, item := range roster {
	fmt.Println(item)
}
```

注意：切片的元素若是 `nil` map，必须先赋一张真正的 map 再写键。`make` 只分配了切片，不会自动给每个元素建表。

## 值为切片类型的map

反过来，值也可以是切片，用来表示“一个键对应多条记录”：

```go
prefs := make(map[string][]string)
prefs["hobby"] = []string{"跑步", "阅读", "素描"}
fmt.Println(prefs)
```

追加时写成 `prefs["hobby"] = append(prefs["hobby"], "游泳")`。键还不存在时，`append` 对 nil 切片也是安全的。

## 示例

统计一句话里每个单词出现了几次。按空格切开，再用 map 累加：

```go
line := "go is fun and go is small"
parts := strings.Split(line, " ")
fmt.Println(parts)
freq := make(map[string]int)
for _, word := range parts {
	freq[word]++
}
fmt.Println(freq)
```

`freq[word]++` 在键不存在时从 0 开始加，所以不必先判断。结果里 `go` 和 `is` 的次数应大于 1，其余单词为 1。
