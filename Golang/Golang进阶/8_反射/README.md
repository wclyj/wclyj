# 反射：运行时查看并修改值

## 它已经藏在常用库里

业务代码很少直接写反射，但标准库经常用它。`json` 能把一段文本填进结构体，靠的就是读取字段标签和类型：

```go
type parcel struct {
	Code string `json:"code"`
	Gram int    `json:"gram"`
}

func main() {
	raw := []byte(`{"code":"HZ-3","gram":480}`)
	var p parcel
	json.Unmarshal(raw, &p)
	fmt.Println(p.Code, p.Gram)
}
```

`Unmarshal` 的第二个参数必须是指针，否则它无法把结果写回去。这和后面“通过反射赋值必须拿到地址”是同一件事。

## 变量里实际有两样东西

- 类型信息：编译期就定下来的元数据，比如字段名、标签、底层种类。
- 值信息：运行过程中会变的那一份数据。

## 反射在解决什么

编译器把变量变成地址之后，名字本身不会留在可执行文件的业务指令里。普通代码在运行时问不出“这个接口里到底装着什么类型”。支持反射的语言会把字段名、类型、结构体布局额外放进可执行文件，并提供一套 API 去读、甚至去改。

Go 的入口是 `reflect`。空接口可以装下任意具体值，但装进去之后静态类型只剩 `interface{}`。反射能在运行时把具体类型和具体值再取出来。

## reflect 怎么拆一个接口值

一个接口值在实现上是“具体类型”加“具体值”。反射把这两部分对应成 `reflect.Type` 和 `reflect.Value`，分别用 `TypeOf` 和 `ValueOf` 取得。

### TypeOf

`TypeOf` 返回类型对象，后面可以问名字、种类、字段和方法。

```go
func showType(x any) {
	fmt.Printf("type:%v\n", reflect.TypeOf(x))
}

func main() {
	showType(float32(2.5))
	showType(int64(48))
}
```

输出：

```text
type:float32
type:int64
```

### 类型名和种类

`Name` 是这段类型在源码里的名字，`Kind` 是它在反射里的大类。用 `type` 新定义的类型，名字和底层种类可以不同；类型别名则和原类型共享名字。数组、切片、映射、指针这些复合类型的 `Name` 是空字符串，因为它们没有单独的定义名。

```go
type meter int64

func showKind(x any) {
	t := reflect.TypeOf(x)
	fmt.Printf("name:%s kind:%s\n", t.Name(), t.Kind())
}

func main() {
	var p *float32
	var m meter
	var r rune
	showKind(p) // name: kind:ptr
	showKind(m) // name:meter kind:int64
	showKind(r) // name:int32 kind:int32

	type crate struct {
		label string
		kg    int
	}
	type note struct{ title string }
	showKind(crate{label: "north", kg: 3})
	showKind(note{title: "库存备忘"})
}
```

`Kind` 是一组整数常量，覆盖布尔、各宽度的整数和浮点、复数、数组、通道、函数、接口、映射、指针、切片、字符串、结构体以及不安全指针。遇到非法的零值 `Value` 时，种类是 Invalid。判断类型时优先用 `Kind`，不要拿 `Name` 和 `"int"` 这种字符串比较，自定义类型的名字对不上。

## ValueOf

`ValueOf` 返回的 `reflect.Value` 带着原始数据，也能再转回普通 Go 值。常用的取出方法：

| 方法 | 作用 |
| --- | --- |
| `Interface()` | 以空接口形式交回，再用类型断言还原 |
| `Int()` | 所有有符号整数都按 `int64` 取出 |
| `Uint()` | 所有无符号整数都按 `uint64` 取出 |
| `Float()` | `float32` 和 `float64` 都按 `float64` 取出 |
| `Bool()` | 取出布尔值 |
| `Bytes()` | 取出字节切片 |
| `String()` | 取出字符串 |

种类和取出方法必须匹配。对一个浮点调用 `Int` 会 panic。

## 按种类读值

```go
func dump(x any) {
	v := reflect.ValueOf(x)
	switch v.Kind() {
	case reflect.Int64:
		fmt.Printf("int64 => %d\n", v.Int())
	case reflect.Float32:
		fmt.Printf("float32 => %f\n", float32(v.Float()))
	case reflect.Float64:
		fmt.Printf("float64 => %f\n", v.Float())
	default:
		fmt.Printf("skip kind %s\n", v.Kind())
	}
}

func main() {
	dump(float32(2.5))
	dump(int64(48))
	boxed := reflect.ValueOf(10)
	fmt.Printf("boxed type %T\n", boxed)
}
```

`boxed` 的静态类型已经是 `reflect.Value`，不再是 `int`。要拿回整数，对它调用 `Int` 或 `Interface`。

## 通过反射改值

函数参数是值拷贝。`ValueOf` 若直接收到整数，得到的是不可寻址的副本，`SetInt` 会 panic。必须传入指针，再用 `Elem` 剥开指针，拿到里面那块可以写的值。

```go
func overwriteCopy(x any) {
	v := reflect.ValueOf(x)
	if v.Kind() == reflect.Int64 {
		v.SetInt(7)
	}
}

func overwrite(x any) {
	v := reflect.ValueOf(x)
	if v.Kind() == reflect.Pointer && v.Elem().Kind() == reflect.Int64 {
		v.Elem().SetInt(7)
	}
}

func main() {
	n := int64(48)
	overwrite(&n)
	fmt.Println(n)
}
```

`CanSet` 可以在写入前先问一句是否允许修改，避免直接 panic。`Elem` 只能用在指针或接口上，对整数调用会 panic。

## IsNil 与 IsValid

### IsNil

`IsNil` 报告通道、函数、接口、映射、指针、切片是否为 nil。用在别的种类上会 panic。它和切片、映射的“长度为 0”不是一回事：空切片不是 nil。

### IsValid

`IsValid` 报告这个 `Value` 里有没有装着东西。零值 `Value` 返回假，除了 `IsValid`、`String`、`Kind` 之外的方法都会 panic。从结构体里找一个不存在的字段，或从映射里取一个不存在的键，得到的就是这种无效值。

```go
func main() {
	var p *int
	fmt.Println("nil pointer", reflect.ValueOf(p).IsNil())
	fmt.Println("bare nil", reflect.ValueOf(nil).IsValid())

	anon := struct{}{}
	fmt.Println("missing field", reflect.ValueOf(anon).FieldByName("code").IsValid())
	fmt.Println("missing method", reflect.ValueOf(anon).MethodByName("Ship").IsValid())

	shelf := map[string]int{}
	key := reflect.ValueOf("no-such")
	fmt.Println("missing key", reflect.ValueOf(shelf).MapIndex(key).IsValid())
}
```

先用 `IsValid` 过滤，再调用 `IsNil` 或 `Int`，顺序反了会在无效值上 panic。

## 结构体

### 相关方法

确认 `Kind` 是结构体之后，可以用类型对象枚举字段：

| 方法 | 作用 |
| --- | --- |
| `Field(i)` | 按下标取字段描述 |
| `NumField` | 字段个数 |
| `FieldByName` | 按名字查找，第二个返回值表示是否找到 |
| `FieldByIndex` | 按一组下标走进嵌套或提升字段 |
| `FieldByNameFunc` | 用自定义函数匹配字段名 |
| `NumMethod` | 方法集里的方法数 |
| `Method(i)` | 方法集中的第 i 个方法 |
| `MethodByName` | 按名字找方法 |

未导出字段可以看见名字和类型，但不能从包外用 `Value` 去改。

### StructField

每个字段的描述里包含：导出名 `Name`、非导出字段所属包路径 `PkgPath`、字段类型、标签、在结构体中的字节偏移、供 `FieldByIndex` 使用的下标路径，以及是否为匿名字段。标签类型上的 `Get` 能按键取出 `json` 或 `ini` 这类标记。

### 遍历字段

```go
type trainee struct {
	Name  string `json:"name"`
	Score int    `json:"score"`
}

func main() {
	one := trainee{Name: "周宁", Score: 86}
	t := reflect.TypeOf(one)
	fmt.Println(t.Name(), t.Kind())
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Printf("name=%s idx=%v type=%v tag=%s\n",
			f.Name, f.Index, f.Type, f.Tag.Get("json"))
	}
	if f, ok := t.FieldByName("Score"); ok {
		fmt.Printf("picked %s tag=%s\n", f.Name, f.Tag.Get("json"))
	}
}
```

### 遍历并调用方法

方法要导出，反射才看得到。值接收者和指针接收者都会影响方法集：`ValueOf` 传入值，就只能看到值接收者方法。

```go
func (s trainee) Review() string {
	msg := "review queued"
	fmt.Println(msg)
	return msg
}

func (s trainee) Rest() string {
	msg := "rest slot booked"
	fmt.Println(msg)
	return msg
}

func listMethods(x any) {
	t := reflect.TypeOf(x)
	v := reflect.ValueOf(x)
	fmt.Println("count", t.NumMethod())
	for i := 0; i < v.NumMethod(); i++ {
		fmt.Printf("name=%s sig=%s\n", t.Method(i).Name, v.Method(i).Type())
		v.Method(i).Call(nil)
	}
}
```

`Call` 的参数是 `[]reflect.Value`。没有参数时传 `nil` 或空切片。返回值同样以 `[]reflect.Value` 交回，需要再用 `String` 或 `Interface` 取出。

## 不要滥用

反射能让框架代码变灵活，但有三个代价：

1. 类型错误要到真正跑到那一行才 panic，离写代码的时间可能很远。
2. 大量反射让调用关系从签名上看不出来，阅读成本高。
3. 反射路径通常比直接调用慢一到两个数量级，不适合放在紧循环里。

能用泛型、接口或代码生成表达的约束，就不要先走到反射。

## 练习：用反射灌进一份 ini

给定这样一段配置：

```ini
[mysql]
address=127.0.0.1
port=3306
username=desk
password=desk
```

再准备一个带 `ini` 标签的结构体。`loadIni` 应接收指针，按标签把等号右边的文本写进字段。整数标签要先转换再 `SetInt`。

```go
type MysqlConf struct {
	Address  string `ini:"address"`
	Port     int    `ini:"port"`
	Username string `ini:"username"`
	Password string `ini:"password"`
}

func loadIni(dst any) {
	v := reflect.ValueOf(dst)
	if v.Kind() != reflect.Pointer {
		panic("dst must be a pointer")
	}
	fmt.Println(v.Elem().Type().Name())
}

func main() {
	var cfg MysqlConf
	loadIni(&cfg)
	fmt.Println(cfg.Address, cfg.Port, cfg.Username, cfg.Password)
}
```

补全时可以逐行扫描：遇到 `[` 开头的行记下区名，遇到 `键=值` 就在结构体里找标签相同的字段。找不到的键跳过，类型对不上要返回错误而不是静默丢掉。
