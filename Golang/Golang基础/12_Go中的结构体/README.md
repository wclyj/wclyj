# Go中的结构体

## 关于结构体

Go 没有类。结构体用来把多份数据放在同一个类型里，角色接近别的语言中的类，但没有继承语法，扩展靠组合和方法集。

单个整数或字符串只能描述事物的一个侧面。要同时保存姓名、年龄和一组爱好，就需要自定义的复合类型，这就是 `struct`。定义之后，它可以像内建类型一样做变量、参数和返回值。

## Type关键字

用 `type` 给一种类型起名。数组和结构体都是值类型：赋值和传参会复制整份数据，这和把对象引用传来传去的语言不同。

### 自定义类型

在已有类型之上可以声明全新的类型。新类型和原来的类型不能直接运算或赋值，即使底层一样。

```go
type myInt int
```

`myInt` 拥有整数的存储方式，但类型名不同，需要显式转换才能和 `int` 混用。

```go
type myInt int

func main() {
	var a myInt = 10
	fmt.Printf("%v %T", a, a)
}
```

输出：

```text
10 main.myInt
```

函数也可以被命名成一种类型，便于把符合签名的函数存进变量：

```go
type myFn func(int, int) int

func add(x int, y int) int {
	return x + y
}

func main() {
	var fn myFn = add
	fmt.Println(fn(1, 2))
}
```

输出：

```bash
3
```

### 类型别名

Go 1.9 起可以用等号声明别名。别名和原类型是同一个类型，只是多了一个名字，类似同一个人的不同称呼。

```go
type TypeAlias = Type
```

`byte` 和 `rune` 就是别名，不是新类型：

```go
type byte = uint8
type rune = int32
```

因此 `byte` 可以直接当作 `uint8` 使用，不必转换。自定义类型则不行。区分二者的标志就是有没有 `=`。

## 结构体定义和初始化

### 结构体的定义

`type` 加 `struct` 列出字段。字段名在本包内可见即可读写；首字母大写的字段还能被其他包访问。

```go
type Person struct {
	name string
	age  int
	sex  string
}

func main() {
	var person Person
	person.name = "张三"
	person.age = 20
	person.sex = "男"
	fmt.Printf("%#v", person)
}
```

类型名本身也遵循同样的可见性：`Person` 能被外部包引用，`person` 这种小写类型名只能留在本包。

对外的数据通常把字段也导出：

```go
type Person struct {
	Name string
	Age  int
	Sex  string
}
```

### 实例化结构体

`var person Person` 得到零值结构体，字符串是空串，整数是 0，然后再逐个字段赋值：

```go
var person Person
person.name = "张三"
person.age = 20
person.sex = "男"
```

### 实例化结构体2

`new` 分配内存并返回指针，字段同样是零值：

```go
person2 := new(Person)
person2.name = "李雷"
person2.age = 31
person2.sex = "女"
fmt.Printf("%#v", person2)
```

打印带 `&`，说明 `person2` 的类型是 `*Person`：

```bash
&main.Person{name:"李雷", age:31, sex:"女"}
```

Go 允许对结构体指针直接写字段，编译器会自动解引用：

```go
person2.name = "李四"
// 等价于
(*person2).name = "李四"
```

### 实例化结构体3

对复合字面量取地址，效果和 `new` 后再赋值类似，但可以同时写出初值。空字面量则得到指向零值的指针：

```go
person3 := &Person{}
person3.name = "赵四"
person3.age = 28
person3.sex = "男"
fmt.Printf("%#v", person3)
```

### 实例化结构体4

字段名和值成对写出，未写出的字段保持零值，顺序可以和声明不同：

```go
person4 := Person{
	name: "张三",
	age:  10,
	sex:  "女",
}
fmt.Printf("%#v", person4)
```

### 实例化结构体5

在键值对前面加 `&`，得到的是指针而不是值：

```go
person5 := &Person{
	name: "孙五",
	age:  10,
	sex:  "女",
}
fmt.Printf("%#v", person5)
```

### 实例化结构体6

省略字段名时，必须按声明顺序给出每一个字段，不能跳过：

```go
person6 := Person{
	"张三",
	5,
	"女",
}
fmt.Println(person6)
```

这种位置写法在字段增减时容易对错位，结构体一长就改用带名字的字面量。

## 结构体方法和接收者

方法是带接收者的函数。接收者写在 `func` 和方法名之间，作用类似其他语言里的 `this`，但命名上不用 `this` 或 `self`。

```go
func (接收者变量 接收者类型) 方法名(参数列表) (返回参数) {
	函数体
}
```

- 接收者变量：官方建议用类型名的首字母，例如 `Person` 用 `p`，读起来短，也不会和字段混淆
- 接收者类型：可以是值或指针
  - 值接收者拿到的是副本，方法里改字段不影响调用方
  - 指针接收者改的是原对象，也避免复制很大的结构体
- 方法名、参数和返回值的规则与普通函数相同

```go
type Person struct {
	name string
	age  int
	sex  string
}

func (p Person) PrintInfo() {
	fmt.Print(" 姓名: ", p.name)
	fmt.Print(" 年龄: ", p.age)
	fmt.Print(" 性别: ", p.sex)
	fmt.Println()
}

func (p *Person) SetInfo(name string, age int, sex string) {
	p.name = name
	p.age = age
	p.sex = sex
}

func main() {
	person := Person{"张三", 18, "女"}
	person.PrintInfo()
	person.SetInfo("李四", 18, "男")
	person.PrintInfo()
}
```

运行结果：

```bash
 姓名: 张三 年龄: 18 性别: 女
 姓名: 李四 年龄: 18 性别: 男
```

`SetInfo` 必须是指针接收者。结构体是值类型，若接收者写成 `Person`，赋值只改副本，`main` 里再打印仍是旧数据。调用时写 `person.SetInfo(...)` 即可，编译器会自动取地址。

## 给任意类型添加方法

接收者不限于结构体。只要是本包里用 `type` 定义的类型，都可以挂方法。不能给别的包的类型，也不能直接给 `int` 这种预声明类型加方法，所以要先做出自定义类型。

```go
type myInt int

func (m myInt) PrintInfo() {
	fmt.Println("自定义类型上的方法")
}

func main() {
	var a myInt = 10
	fmt.Printf("%v %T\n", a, a)
	a.PrintInfo()
}
```

## 结构体的匿名字段

字段可以只写类型、不写名字，这种字段叫匿名字段。它的名字默认就是类型名，因此同一种类型的匿名字段在一个结构体里只能出现一次，否则名字冲突。

```go
type Person struct {
	string
	int
}

func main() {
	person := Person{"张三", 18}
	fmt.Println(person.string, person.int)
}
```

外层用类型名访问，例如 `person.string`。匿名字段更常见的用途是嵌入另一个结构体，从而把它的字段和方法提升到外层。

字段类型可以是基本类型，也可以是切片、map 或另一个结构体。指针、切片和 map 的零值是 `nil`，还没有可用存储。要写入元素，得先 `make` 或赋一个非空字面量。

```go
type Person struct {
	name     string
	age      int
	hobby    []string
	mapValue map[string]string
}

func main() {
	var person Person
	person.name = "张三"
	person.age = 10

	person.hobby = make([]string, 3)
	person.hobby[0] = "睡觉"
	person.hobby[1] = "吃饭"
	person.hobby[2] = "散步"

	person.mapValue = make(map[string]string)
	person.mapValue["address"] = "北京"
	person.mapValue["phone"] = "123456789"

	fmt.Printf("%#v", person)
}
```

结构体也可以嵌套具名结构体，用来表达“拥有”而不是“是一种”：

```go
type User struct {
	userName string
	password string
	sex      string
	age      int
	address  Address
}

type Address struct {
	name  string
	phone string
	city  string
}

func main() {
	var u User
	u.userName = "notebook"
	u.password = "secret"
	u.sex = "男"
	u.age = 18

	u.address = Address{
		name:  "张三",
		phone: "110",
		city:  "北京",
	}
	fmt.Printf("%#v", u)
}
```

外层通过 `u.address.city` 逐层访问。这里 `address` 是具名字段，不会把 `city` 直接提升到 `User`。

## 嵌套结构体的字段名冲突

内层和外层若有同名字段，直接写字段名时优先用外层。外层没有，才提升内层的字段。若两个被嵌入的结构体都有这个名字，而外层又没有，编译器无法决定选哪一个，必须写成 `外层.内层.字段`。

读和写都遵守同一规则。不要依赖提升去改“可能来自某一层”的字段，歧义时应写全路径。

## 结构体的继承

Go 没有 `extends`。把另一个结构体作为匿名字段嵌进去，外层就得到内层的字段和方法，效果接近组合式的复用。

```go
type Animal struct {
	name string
}

func (a Animal) run() {
	fmt.Printf("%v 在运动\n", a.name)
}

type Dog struct {
	age int
	Animal
}

func (dog Dog) wang() {
	fmt.Printf("%v 在叫\n", dog.name)
}

func main() {
	dog := Dog{
		age: 10,
		Animal: Animal{
			name: "阿帕奇",
		},
	}
	dog.run()
	dog.wang()
}
```

运行结果：

```bash
阿帕奇 在运动
阿帕奇 在叫
```

`run` 仍定义在 `Animal` 上，`Dog` 因为嵌入了它，方法集里就能直接调用。这不是子类覆盖父类：若 `Dog` 自己再定义 `run`，则外层方法优先，内层方法要通过 `dog.Animal.run()` 调用。

## Go中的结构体和Json相互转换

JSON 是一种文本数据格式，人和程序都容易读写。接口常用它交换数据，例如：

```json
{
    "name": "张三",
    "age": 15
}
```

给客户端返回数据时，要把结构体变成 JSON 文本，这叫序列化；把收到的文本变回结构体，叫反序列化。标准库 `encoding/json` 提供 `Marshal` 和 `Unmarshal`。

```go
type Student struct {
	ID     string
	Gender string
	Name   string
	Sno    string
}

func main() {
	s1 := Student{
		ID:     "12",
		Gender: "男",
		Name:   "李四",
		Sno:    "s001",
	}
	raw, _ := json.Marshal(s1)
	fmt.Printf("%s", raw)
}
```

`Marshal` 返回 `[]byte`。反序列化时要传入指针，函数才能把结果写回变量：

```go
type Student struct {
	ID     string
	Gender string
	Name   string
	Sno    string
}

func main() {
	text := `{"ID":"12","Gender":"男","Name":"李四","Sno":"s001"}`
	var s2 Student
	err := json.Unmarshal([]byte(text), &s2)
	if err != nil {
		fmt.Println("转换失败")
	} else {
		fmt.Printf("%#v\n", s2)
	}
}
```

### 注意

只有导出字段才会进入 JSON。首字母小写的字段对本包可见，但对 `encoding/json` 不可见，序列化时会被丢掉，反序列化时也不会被填充。准备交给 JSON 的结构体，字段名要大写。

## 结构体标签Tag

标签是写在字段后面的元数据，运行时用反射读取。它包在反引号里，形式是空格分隔的键值对：

```text
key1:"value1" key2:"value2"
```

键和值之间用冒号，值用双引号。解析很严格：键和值之间多一个空格，编译不会报错，但反射读不到，JSON 名字会静默退回字段名。不要写成 `json: "id"` 这种带空格的形式。

用 `json` 标签可以把对外的键改成小写或别的名字：

```go
type Student2 struct {
	Id     string `json:"id"`
	Gender string `json:"gender"`
	Name   string `json:"name"`
	Sno    string `json:"sno"`
}

func main() {
	s1 := Student2{Id: "12", Gender: "男", Name: "李四", Sno: "s001"}
	raw, _ := json.Marshal(s1)
	fmt.Println(string(raw))

	text := `{"id":"12","gender":"男","name":"李四","sno":"s001"}`
	var s2 Student2
	err := json.Unmarshal([]byte(text), &s2)
	if err != nil {
		fmt.Println("转换失败")
	} else {
		fmt.Printf("%#v\n", s2)
	}
}
```

序列化结果的键是 `id` 而不是 `Id`。反序列化也要使用标签里的名字；若文本仍用 `Id` 作为键，字段对不上，`s2` 会保持零值。

## 嵌套结构体和Json序列化反序列化

嵌套结构体和切片字段同样走 `Marshal`。切片要用空切片而不是 `nil`，这样 JSON 里是 `[]` 而不是 `null`；下面的循环会再往里追加元素。

```go
type Student3 struct {
	Id     int
	Gender string
	Name   string
}

type Class struct {
	Title    string
	Students []Student3
}

func main() {
	class := Class{
		Title:    "1班",
		Students: make([]Student3, 0),
	}
	for i := 0; i < 10; i++ {
		class.Students = append(class.Students, Student3{
			Id:     i + 1,
			Gender: "男",
			Name:   fmt.Sprintf("stu_%v", i+1),
		})
	}
	fmt.Printf("%#v\n", class)

	raw, err := json.Marshal(class)
	if err != nil {
		fmt.Println("序列化失败")
	} else {
		fmt.Println(string(raw))
	}
}
```

反序列化嵌套数据时，把整段 JSON 交给 `Unmarshal`，并传入 `*Class`。内层切片会按数组长度自动分配，不需要事先 `make`。字段仍然必须导出，嵌套类型的字段也不例外。
