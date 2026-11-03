# 单元测试、基准测试与示例

不写测试的改动很难证明自己没有碰坏原有行为。测试驱动开发把失败的测试写在实现前面。国内很多项目把测试留到上线前才补，这一节只讨论 Go 里怎么把单元测试和基准测试写进仓库，不讨论流程口号。

## go test 怎么找到测试

测试不需要新的语法。`go test` 在包目录里收集以 `_test.go` 结尾的文件，这些文件不会被 `go build` 打进最终二进制。

文件里有三类特殊函数：

| 种类 | 名字 | 用途 |
| --- | --- | --- |
| 单元测试 | 以 `Test` 开头 | 判断行为是否符合预期 |
| 基准测试 | 以 `Benchmark` 开头 | 测量耗时和分配 |
| 示例 | 以 `Example` 开头 | 既当文档，也能对照输出 |

`go test` 会生成一个临时的 main 去调用它们，跑完删除临时文件，再把通过或失败打印出来。

## 测试函数的样子

必须导入 `testing`。签名固定为接收 `*testing.T`：

```go
func TestName(t *testing.T) {
}
```

`Test` 后面如果还有后缀，后缀要以大写字母开头，例如 `TestCarve`、`TestCarveEmpty`。`Testcarve` 不会被识别。

`T` 上常用的方法可以分成几组：

- `Error` / `Errorf` 记录失败并继续执行。
- `Fatal` / `Fatalf` 记录失败并立刻返回。
- `Log` / `Logf` 只在失败或打开 `-v` 时显示。
- `Run` 启动一个有名字的子测试。
- `Parallel` 标记该测试可与其它同样标记的测试并行。
- `Skip` / `SkipNow` 跳过，不记为失败。
- `Failed` 报告当前是否已经失败。

失败信息要写出输入、期望和实际值。只写“不对”会让下次失败时无从查起。

## 用一个切分函数串起流程

被测代码放在 `carve.go`：

```go
package carve

import "strings"

// Carve 用 knife 把 text 切开，保留空段。
func Carve(text, knife string) []string {
	out := make([]string, 0)
	for {
		i := strings.Index(text, knife)
		if i < 0 {
			return append(out, text)
		}
		out = append(out, text[:i])
		text = text[i+1:]
	}
}
```

旁边放 `carve_test.go`：

```go
package carve

import (
	"reflect"
	"testing"
)

func TestCarvePipe(t *testing.T) {
	got := Carve("red|green|blue", "|")
	want := []string{"red", "green", "blue"}
	if !reflect.DeepEqual(got, want) {
		t.Errorf("want %v got %v", want, got)
	}
}
```

在包目录执行 `go test`，通过时类似：

```text
PASS
ok      example.com/lab/carve   0.012s
```

再加一个多字符分隔符的用例：

```go
func TestCarveWord(t *testing.T) {
	got := Carve("keepMEout", "ME")
	want := []string{"keep", "out"}
	if !reflect.DeepEqual(got, want) {
		t.Errorf("want %v got %v", want, got)
	}
}
```

这一次会失败。`-v` 能看到每个函数的名字和耗时：

```text
=== RUN   TestCarvePipe
--- PASS: TestCarvePipe (0.00s)
=== RUN   TestCarveWord
--- FAIL: TestCarveWord (0.00s)
    carve_test.go:18: want [keep out] got [keep Eout]
FAIL
```

`-run` 后面是正则，只有名字匹配的测试才会跑：

```bash
go test -v -run Word
```

失败的原因是切完后只跳过了 1 个字节，分隔符 `ME` 有两个字节，第二个字符被留在了下一段。应按分隔符的长度前进：

```go
func Carve(text, knife string) []string {
	out := make([]string, 0)
	for {
		i := strings.Index(text, knife)
		if i < 0 {
			return append(out, text)
		}
		out = append(out, text[:i])
		text = text[i+len(knife):]
	}
}
```

改完不要只重跑刚才失败的那个。完整跑一遍，确认修复没有弄坏单字符的情况：

```bash
go test -v
```

两边都通过，才算这次修改收尾。

## 表格驱动

还想覆盖中文和“分隔符根本不存在”的情况。与其再复制一个函数，不如把输入和期望放进切片。

```go
func TestCarve(t *testing.T) {
	type scene struct {
		text  string
		knife string
		want  []string
	}
	scenes := []scene{
		{"red|green|blue", "|", []string{"red", "green", "blue"}},
		{"red|green|blue", ",", []string{"red|green|blue"}},
		{"keepMEout", "ME", []string{"keep", "out"}},
		{"北风吹雪", "风", []string{"北", "吹雪"}},
	}
	for _, sc := range scenes {
		got := Carve(sc.text, sc.knife)
		if !reflect.DeepEqual(got, sc.want) {
			t.Errorf("want %#v got %#v", sc.want, got)
		}
	}
}
```

若期望写错，例如以为开头就是分隔符时不会留下空串：

```text
want []string{"风吹雪"} got []string{"北", "风吹雪"}
```

这里故意把期望写成错误的，是为了展示失败信息。`"北风吹雪"` 用 `"风"` 切开，第一段是 `"北"`，不是空串。真正会产生空串的是分隔符出现在开头，例如 `"风吹雪"` 配 `"风"`，结果是 `[]string{"", "吹雪"}`。

`%v` 打印切片时，空字符串几乎看不见。失败信息改用 `%#v`，能看出有没有空串：

```go
t.Errorf("want %#v got %#v", sc.want, got)
```

输出会变成 Go 语法：

```text
want []string{"吹雪"} got []string{"", "吹雪"}
```

## 子测试

用例多了以后，一条 `Errorf` 看不出是哪一组。可以给每组起名字，写进错误文本：

```go
scenes := map[string]scene{
	"pipes":   {"red|green|blue", "|", []string{"red", "green", "blue"}},
	"no hit":  {"red|green|blue", ",", []string{"red|green|blue"}},
	"word":    {"keepMEout", "ME", []string{"keep", "out"}},
	"prefix":  {"风吹雪", "风", []string{"", "吹雪"}},
}
for name, sc := range scenes {
	got := Carve(sc.text, sc.knife)
	if !reflect.DeepEqual(got, sc.want) {
		t.Errorf("%s want %#v got %#v", name, sc.want, got)
	}
}
```

Go 1.7 起更直接的方式是 `t.Run`。每个子测试有自己的通过或失败行：

```go
for name, sc := range scenes {
	sc := sc
	t.Run(name, func(t *testing.T) {
		got := Carve(sc.text, sc.knife)
		if !reflect.DeepEqual(got, sc.want) {
			t.Errorf("want %#v got %#v", sc.want, got)
		}
	})
}
```

循环变量在 Go 1.22 之前被闭包共享，所以这里把 `sc` 再赋值一次。1.22 及以后每次迭代已有独立变量，这行变成多余但无害。

失败时输出形如：

```text
--- FAIL: TestCarve (0.00s)
    --- FAIL: TestCarve/prefix (0.00s)
        carve_test.go:40: want []string{"吹雪"} got []string{"", "吹雪"}
    --- PASS: TestCarve/pipes (0.00s)
```

期望修正为带前导空串之后，子测试全部通过。`-run` 可以用斜杠选中子测试：

```bash
go test -v -run 'Carve/pipes'
```

## 覆盖率

覆盖率是测试至少执行过一次的语句占全部语句的比例。它不知道断言是否有意义，只知道哪些行被走到了。

```bash
go test -cover
```

```text
PASS
coverage: 100.0% of statements
ok      example.com/lab/carve   0.008s
```

加上 `-coverprofile` 把计数写到文件：

```bash
go test -cover -coverprofile=cover.out
go tool cover -html=cover.out
```

第二条会生成一份 HTML。执行过的语句和没执行过的语句用不同颜色标出。没有浏览器时，`go tool cover -func=cover.out` 会按函数打印百分比。100% 只说明每条语句都跑过，不说明边界都断言了。

## 基准测试

基准测试看的是工作负载下的耗时。函数名以 `Benchmark` 开头，参数是 `*testing.B`，循环必须跑满 `b.N` 次。`b.N` 由测试框架从 1 开始按 1、2、5、10 这样的序列增加，直到这一轮稳定、并且默认至少跑够约 1 秒。

`B` 上除了和 `T` 类似的失败方法，还有：

- `ResetTimer` / `StartTimer` / `StopTimer` 把准备工作和计时分开。
- `ReportAllocs` 记录每次操作的分配。
- `SetBytes` 让报告按字节数折算吞吐。
- `RunParallel` 在多个 goroutine 里分摊 `b.N`。
- `Run` 写出有名字的子基准。

为 `Carve` 写一个基准：

```go
func BenchmarkCarve(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Carve("北风吹雪又吹沙", "吹")
	}
}
```

默认 `go test` 不跑基准。加上 `-bench`，它的值也是正则：

```bash
go test -bench=Carve
```

```text
goos: linux
goarch: amd64
pkg: example.com/lab/carve
BenchmarkCarve-8        8000000               190 ns/op
PASS
```

`BenchmarkCarve-8` 里的 8 是 `GOMAXPROCS`。`8000000` 是总次数，`190 ns/op` 是平均每次调用的耗时。

`-benchmem` 追加内存统计：

```bash
go test -bench=Carve -benchmem
```

```text
BenchmarkCarve-8        8000000               188 ns/op              96 B/op          4 allocs/op
```

`96 B/op` 是平均每次分配的字节数，`4 allocs/op` 是分配次数。`Carve` 每次 `append` 都可能扩容。若事先按分隔符出现次数分配容量，扩容会少很多：

```go
func Carve(text, knife string) []string {
	out := make([]string, 0, strings.Count(text, knife)+1)
	for {
		i := strings.Index(text, knife)
		if i < 0 {
			return append(out, text)
		}
		out = append(out, text[:i])
		text = text[i+len(knife):]
	}
}
```

再测一次，分配次数和字节数应明显下降，耗时往往也会一起降。优化前后要使用同一条命令，机器负载不同时绝对数字不能直接对比。

## 比较不同输入规模

单个基准只给出绝对耗时。要回答“规模扩大十倍慢多少”或“两种算法谁更快”，应把规模写成参数，用多个基准函数调用同一份实现。

```go
func benchWays(b *testing.B, n int) {
	for i := 0; i < b.N; i++ {
		Ways(n)
	}
}

func BenchmarkWays2(b *testing.B)  { benchWays(b, 2) }
func BenchmarkWays5(b *testing.B)  { benchWays(b, 5) }
func BenchmarkWays10(b *testing.B) { benchWays(b, 10) }
func BenchmarkWays20(b *testing.B) { benchWays(b, 20) }
func BenchmarkWays35(b *testing.B) { benchWays(b, 35) }
```

被测函数是爬楼梯：一次走 1 级或 2 级，问走到第 n 级有多少种走法。它的递归形态和斐波那契一样，用来观察指数增长：

```go
func Ways(n int) int {
	if n < 2 {
		return 1
	}
	return Ways(n-1) + Ways(n-2)
}
```

```bash
go test -bench=Ways
```

随着 n 变大，`ns/op` 会急剧上升。`BenchmarkWays35` 可能只跑了几次，因为默认 1 秒的预算很快被一次调用吃完。用 `-benchtime` 拉长预算，次数变多，平均值更稳：

```bash
go test -bench=Ways35 -benchtime=20s
```

不要把 `b.N` 当成问题规模。下面两种写法都错了：第一种每次循环都在算不同的 n，报告的平均值没有意义；第二种直接用 `b.N` 当参数，框架把 `b.N` 调大时问题本身也变难，时间不再线性。

```go
func BenchmarkWaysWrong(b *testing.B) {
	for n := 0; n < b.N; n++ {
		Ways(n)
	}
}

func BenchmarkWaysWrong2(b *testing.B) {
	Ways(b.N)
}
```

规模应像 `benchWays` 那样固定，只让 `b.N` 决定重复次数。

## 把准备时间排除在外

`ResetTimer` 之前的耗时不计入报告。构造大输入、预热缓存，都应放在它前面。

```go
func BenchmarkCarveReset(b *testing.B) {
	time.Sleep(200 * time.Millisecond)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		Carve("北风吹雪又吹沙", "吹")
	}
}
```

循环内部若有不该计时的动作，用 `StopTimer` 和 `StartTimer` 包住。不要在计时区间里打印日志。

## 并行基准

`RunParallel` 启动多个 goroutine，把 `b.N` 分给它们。默认并行度是 `GOMAXPROCS`。若瓶颈不在 CPU，可以先 `SetParallelism` 提高倍数。它常和 `-cpu` 一起用，用来对比 1 个核与多个核。

```go
func BenchmarkCarveParallel(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			Carve("北风吹雪又吹沙", "吹")
		}
	})
}
```

```bash
go test -bench=Carve -cpu 1,4
```

并行结果更低，只说明这个函数能扩展，不说明业务里就该开这么多 goroutine。函数如果改共享变量，并行基准还会变成数据竞争，要先用 `-race` 看过。

## 测试前后的准备和清理

有的资源要在全部测试之前打开，全部结束之后关掉。在 `_test.go` 里定义 `TestMain`：

```go
func TestMain(m *testing.M) {
	fmt.Println("open fixtures")
	code := m.Run()
	fmt.Println("close fixtures")
	os.Exit(code)
}
```

生成的测试会先进入 `TestMain`。`m.Run` 才真正执行各个 `Test`。退出码必须传给 `os.Exit`，否则失败也会被显示成进程成功。

调用 `TestMain` 时，测试框架还没解析命令行。若测试依赖 `-coverprofile` 这类标志，要在 `m.Run` 之前显式调用 `flag.Parse()`。

### 每一组、每一条子测试

准备和清理也可以下放到单个测试或子测试。返回一个函数，交给 `defer`，失败时也会执行。

```go
func readyCase(t *testing.T) func(t *testing.T) {
	t.Log("case ready")
	return func(t *testing.T) {
		t.Log("case closed")
	}
}

func readySub(t *testing.T) func(t *testing.T) {
	t.Log("sub ready")
	return func(t *testing.T) {
		t.Log("sub closed")
	}
}

func TestCarveLifecycle(t *testing.T) {
	scenes := map[string]struct {
		text, knife string
		want        []string
	}{
		"pipes":  {"red|green|blue", "|", []string{"red", "green", "blue"}},
		"no hit": {"red|green|blue", ",", []string{"red|green|blue"}},
		"word":   {"keepMEout", "ME", []string{"keep", "out"}},
		"prefix": {"风吹雪", "风", []string{"", "吹雪"}},
	}
	done := readyCase(t)
	defer done(t)
	for name, sc := range scenes {
		sc := sc
		t.Run(name, func(t *testing.T) {
			subDone := readySub(t)
			defer subDone(t)
			got := Carve(sc.text, sc.knife)
			if !reflect.DeepEqual(got, sc.want) {
				t.Errorf("want %#v got %#v", sc.want, got)
			}
		})
	}
}
```

`-v` 时能看到：外层的准备最先出现，每个子测试的准备和清理成对出现，外层清理在所有子测试之后。`t.Log` 在通过且没有 `-v` 时不显示，所以排查顺序时要带上 `-v`。

## 示例函数

第三种特殊函数以 `Example` 开头，没有参数也没有返回值。

```go
func ExampleCarve() {
	fmt.Println(Carve("red|green|blue", "|"))
	fmt.Println(Carve("风吹雪", "风"))
	// Output:
	// [red green blue]
	// [ 吹雪]
}
```

它有三个用途：

1. 文档工具会把示例挂到对应的函数下面，读者不用翻测试文件。
2. 注释里写了 `Output:` 时，`go test` 会执行函数并逐行对比标准输出。不一致则失败。

```bash
go test -run Example
```

3. 示例是可以直接运行的最小程序。公开文档站点上，读者能在页面里改两行再执行。对比时注意空格：`[ 吹雪]` 开头有一个空格，因为第一段是空字符串，`fmt.Println` 打印切片时用空格连接。

没有 `Output:` 的示例只作为文档编译，不会被拿来对比。输出不稳定的函数（带时间、带随机数）不要写死 `Output:`，可以改用 `Unordered output:` 忽略行顺序，或干脆不断言。

## Error 和 Fatal 怎么选

`Errorf` 记下失败后还会继续执行后面的断言，适合一次函数调用里要核对多个字段。`Fatalf` 立刻返回，适合前置条件已经不成立、后面的断言没有意义的情况。例如打开夹具失败就不必再比输出。

```go
func TestLoad(t *testing.T) {
	raw, err := os.ReadFile("testdata/sample.txt")
	if err != nil {
		t.Fatalf("fixture: %v", err)
	}
	if !bytes.Contains(raw, []byte("red")) {
		t.Errorf("sample missing red: %q", raw)
	}
}
```

夹具放在包下的 `testdata` 目录。`go` 工具默认忽略这个目录名，不会把它当成要编译的包。期望输出很长时，可以把整段文本放进 `testdata`，测试里读出来再比较，避免断言字符串把测试文件撑得无法阅读。

## 辅助函数

重复的比较可以抽成小函数。失败时用 `t.Helper()`，报告的行号会落在调用处，而不是辅助函数内部。

```go
func equalParts(t *testing.T, got, want []string) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("want %#v got %#v", want, got)
	}
}
```

不要在辅助函数里启动新的 `t.Run` 却忘了把 `t` 传进去。子测试必须使用 `Run` 回调拿到的那个 `t`，否则失败会计到父测试上，`-run` 也选不中。

## 竞态

只靠功能测试看不出并发写切片、并发写映射。加上 `-race` 后，运行时会记录每次内存访问，发现没有同步的冲突就让测试失败：

```bash
go test -race -count=1
```

`-count=1` 禁用结果缓存，避免上次的通过被直接复用。竞态检测会明显变慢，适合在提交前跑，而不是每次改一行都跑。基准测试一般不要加 `-race`，检测本身的开销会淹没被测函数的耗时。

## 子基准

和子测试一样，`b.Run` 能在一次进程里对比几种实现，报告按名字分开：

```go
func BenchmarkCarveSizes(b *testing.B) {
	samples := map[string]string{
		"short": "红|绿",
		"long":  strings.Repeat("红|绿|", 40),
	}
	for name, text := range samples {
		text := text
		b.Run(name, func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				Carve(text, "|")
			}
		})
	}
}
```

对比两个实现时，输入必须相同。一个用短字符串、一个用长字符串，得出的快慢不能说明算法差异。

## 把回文判断写成可测的函数

练习可以先落成下面这个形状。`Mirror` 去掉空白和标点、转成小写后再比较两端。这里用简体句子和一句英文同时覆盖。

```go
func Mirror(s string) bool {
	var b strings.Builder
	for _, r := range s {
		if unicode.IsLetter(r) || unicode.IsDigit(r) {
			b.WriteRune(unicode.ToLower(r))
		}
	}
	body := b.String()
	rs := []rune(body)
	for i, j := 0, len(rs)-1; i < j; i, j = i+1, j-1 {
		if rs[i] != rs[j] {
			return false
		}
	}
	return true
}
```

按码点而不是按字节对折，否则中文的三个字节会被拆开，两端永远对不上。表格里至少放这些行：空串、单字符、`油灯少灯油`、带逗号的英文、以及一个明确不是回文的词。基准测试看的是 `Builder` 和两次扫描的分配；若 `-benchmem` 显示每次都有多块堆分配，再考虑把缓冲放进调用方传入的 `sync.Pool`，但先确认功能测试全绿。

## 常见失误

- 测试和被测函数不在同一个包，又去访问未导出的辅助逻辑，编译直接失败。需要黑盒视角时把测试包名写成 `carve_test`，只能调用导出的名字。
- 用 `==` 比较切片或映射，编译器会拒绝。切片用 `reflect.DeepEqual`，或在 Go 1.21 之后用 `slices.Equal`。
- 断言写反，把实际值放在“期望”的位置，失败信息会把人带偏。约定参数顺序是期望在前、实际在后。
- 缓存了通过的测试后改了代码却没改测试文件，`go test` 可能不重新编译。怀疑时加 `-count=1`。
- 基准函数里若编译器发现结果没人使用，可能把调用整个删掉。用一个包级变量接住返回值，防止优化掉被测代码。

```go
var sink []string

func BenchmarkCarveSink(b *testing.B) {
	var got []string
	for i := 0; i < b.N; i++ {
		got = Carve("北风吹雪又吹沙", "吹")
	}
	sink = got
}
```

黑盒测试包名写成 `carve_test` 时，不能调用未导出的 `Carve` 内部帮手，只能走导出 API。这样能防止测试依赖实现细节，重构内部时测试不用跟着改。需要覆盖未导出分支时，把测试留在同一个包里。

## 练习

写一个判断回文的函数：忽略大小写、空格和标点后，正着读和反着读相同。例如 `Madam, I'm Adam` 和 `油灯少灯油` 都算。为它补表格测试和基准测试，再根据分配次数决定要不要事先准备缓冲区。中英文混排、空字符串和单个字符要各有一条用例。
