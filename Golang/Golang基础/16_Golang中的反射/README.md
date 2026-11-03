# Go中的反射

## 反射

有时一个函数要同时接受多种具体类型，这些类型没有共同接口，布局在写函数时还不知道，甚至类型本身是后来才加上的。这时要在运行时查看值的类型和内容，就用反射。

空接口能装下任何值，但装进去之后静态类型只剩 `interface{}`。要知道里面原来是什么，有两条路：

- 类型断言，适合候选类型很少、事先能列出来的情况
- 反射，在运行时取出类型信息和当前值

把结构体编成 JSON 时，字段名和标签就是靠反射读出来的。后面 ORM 把结构体字段对应到列，底层同样是在运行时查看类型。

对象关系映射指的是：用一份描述对象与表之间对应关系的元数据，把内存里的对象写进关系数据库，或把行读回对象。

## 反射的基本介绍

反射是程序在运行期间检查并修改自身数据的能力。普通编译会把变量变成地址，名字不会留在可执行文件里，因此运行中的程序默认不知道“这个字段叫什么”。支持反射的语言会把字段名、类型、方法列表等元数据放进二进制，并提供一组 API 去读、去改。

## Go可以实现的功能

- 运行时读取变量的类型和种类
- 若是结构体，还能列出字段、标签和方法
- 在满足可设置的前提下修改值，或按名字调用方法

一个变量在反射里分成两部分：

- 类型信息：编译期就定下的元数据
- 值信息：运行过程中会变化的那一份数据

接口值同样由“具体类型 + 具体值”组成。`reflect` 包把这两部分表示成 `reflect.Type` 和 `reflect.Value`，入口函数是 `TypeOf` 和 `ValueOf`。

## reflect.TypeOf()获取任意值的类型对象

`reflect.TypeOf` 接受空接口，返回 `reflect.Type`。通过这个对象可以问名字、种类、字段个数，而不必在编译期写死类型。

```go
func showType(x interface{}) {
	t := reflect.TypeOf(x)
	fmt.Println(t)
}

func main() {
	showType(3)
	showType(1.5)
	showType("go")
	showType(false)
}
```

分别会看到 `int`、`float64`、`string`、`bool`。传入的是具体值，函数参数才是空接口；`TypeOf` 看到的是装进去之前的动态类型。

## type name 和 type Kind

反射里要区分类型名和种类。`type` 可以造出很多新名字，种类表示底层大类，例如指针、结构体、切片、整数。自定义类型 `type Score int` 的名字是 `Score`，种类仍是 `int`。

数组、切片、map、指针的 `Name()` 往往是空字符串，因为它们是复合构造，没有单独的定义名。要分支处理时看 `Kind()`，不要只看 `Name()`。

```go
t := reflect.TypeOf(x)
fmt.Println("类型 ", t)
fmt.Println("类型名称 ", t.Name())
fmt.Println("类型种类 ", t.Kind())
```

以前用断言把空接口里的整数取出来再运算：

```go
func addAssert(x interface{}) {
	n, _ := x.(int)
	fmt.Println(10 + n)
}
```

断言失败会得到零值，错误被丢掉。改成反射后可以先看种类，再取值：

```go
func addReflect(x interface{}) {
	v := reflect.ValueOf(x)
	fmt.Println(v)
	fmt.Println(v.Int() + 7)
}
```

`Int()` 要求种类是有符号整数，否则会 panic。更稳妥的写法是先 `switch` 种类：

```go
v := reflect.ValueOf(x)
switch v.Kind() {
case reflect.Int:
	fmt.Println("signed int")
case reflect.Float64:
	fmt.Println("float64")
default:
	fmt.Println("other kind")
}
```

## reflect.ValueOf

`ValueOf` 返回 `reflect.Value`，里面包着当前的值。它和原始变量可以互转：`Interface()` 变回空接口，再断言成具体类型；整数、无符号数和浮点也有直接取值的方法。

| 方法 | 说明 |
| --- | --- |
| `Interface()` | 以空接口形式返回，再用断言还原 |
| `Int() int64` | 有符号整数都走这里 |
| `Uint() uint64` | 无符号整数都走这里 |
| `Float() float64` | `float32` 与 `float64` 都按双精度返回 |

取值方法必须和种类匹配。想修改原变量时，传入的应当是指针，再 `Elem()` 得到可设置的 `Value`，否则 `CanSet()` 为 false。

## 结构体反射

### 与结构体相关的方法

当 `Kind` 是结构体时，`reflect.Type` 可以按序号或名字取字段。常用方法：

| 方法 | 说明 |
| --- | --- |
| `Field(i int) StructField` | 按下标取字段元数据 |
| `NumField() int` | 字段个数 |
| `FieldByName(name string) (StructField, bool)` | 按名字查找，第二个结果表示是否找到 |
| `FieldByIndex(index []int) StructField` | 用一串下标走进嵌套结构体 |

`StructField` 上能读到名字、类型和标签。标签字符串要用 `Tag.Get` 按键取出，格式必须是反引号里的 `key:"value"`。

```go
type Learner struct {
	Name  string `json:"name"`
	Age   int    `json:"age"`
	Score int    `json:"score"`
}

func (s Learner) Summary() string {
	return fmt.Sprintf("name=%v age=%v score=%v", s.Name, s.Age, s.Score)
}

func (s *Learner) Reset(name string, age int, score int) {
	s.Name = name
	s.Age = age
	s.Score = score
}

func (s Learner) Dump() {
	fmt.Println("dump learner")
}

func dumpFields(s interface{}) {
	t := reflect.TypeOf(s)
	if t.Kind() != reflect.Struct {
		fmt.Println("need a struct value")
		return
	}

	f0 := t.Field(0)
	fmt.Printf("%#v\n", f0)
	fmt.Println("name:", f0.Name)
	fmt.Println("type:", f0.Type)
	fmt.Println("tag:", f0.Tag.Get("json"))

	if f1, ok := t.FieldByName("Age"); ok {
		fmt.Println("name:", f1.Name)
		fmt.Println("type:", f1.Type)
		fmt.Println("tag:", f1.Tag)
	}

	fmt.Println("field count:", t.NumField())

	v := reflect.ValueOf(s)
	fmt.Println("name value:", v.FieldByName("Name"))
}

func main() {
	item := Learner{"lin", 20, 88}
	dumpFields(item)
}
```

传入指针时，`Kind` 是 `Ptr`，要先 `t.Elem()` 才是结构体。上面的判断只接受值，避免在指针上直接 `Field` 导致 panic。

读方法、再按名字调用，使用的是值和类型两侧的方法表。值接收者和指针接收者是否出现，取决于你传入的是值还是指针。

```go
func callMethods(s interface{}) {
	t := reflect.TypeOf(s)
	if t.Kind() != reflect.Ptr || t.Elem().Kind() != reflect.Struct {
		fmt.Println("need a pointer to struct")
		return
	}

	m0 := t.Method(0)
	fmt.Println(m0.Name)
	fmt.Println("method count:", t.NumMethod())

	v := reflect.ValueOf(s)
	v.MethodByName("Dump").Call(nil)

	args := []reflect.Value{
		reflect.ValueOf("wu"),
		reflect.ValueOf(21),
		reflect.ValueOf(90),
	}
	v.MethodByName("Reset").Call(args)
	v.MethodByName("Dump").Call(nil)
}
```

`Method` 的顺序按方法名排序，不是按源码书写顺序。`Call` 的参数是 `[]reflect.Value`，没有参数时传 `nil`。`Reset` 是指针接收者，所以这里必须传入 `*Learner`，否则方法集合里找不到它。调用前最好用 `MethodByName` 的第二个返回值确认方法存在。
