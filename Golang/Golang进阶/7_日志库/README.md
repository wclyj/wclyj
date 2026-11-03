# 标准日志与一个分级记录器

## 为什么要先把日志留住

调试阶段靠它还原执行路径，上线之后靠它追查某次请求为什么失败。Go 的标准库 `log` 提供了最小可用的实现：带时间戳，能选择输出到终端或文件。下面先用它，再写一个带级别开关的简化版本。

## 直接使用包级函数

`log` 里有一个现成的 `Logger`，不用自己构造。三类函数的差别在写完之后的动作：

- `Print`、`Printf`、`Println` 只写日志。
- `Fatal`、`Fatalf`、`Fatalln` 写完后以状态码 1 结束进程。
- `Panic`、`Panicf`、`Panicln` 写完后触发 panic。

```go
package main

import "log"

func main() {
	log.Println("heartbeat from billing")
	kind := "audit"
	log.Printf("channel=%s ready\n", kind)
	log.Fatalln("ledger file missing")
	log.Panicln("this line is never reached")
}
```

`Fatalln` 会让进程退出，所以最后一行不会执行。默认输出到标准错误，每条前面带日期和时间：

```text
2024/04/02 09:15:01 heartbeat from billing
2024/04/02 09:15:01 channel=audit ready
2024/04/02 09:15:01 ledger file missing
```

## 改写到文件

长期运行的服务不能只打在终端上。`SetOutput` 接受任何 `io.Writer`，把文件打开后交出去即可。

```go
func main() {
	fp, err := os.OpenFile("./billing.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		fmt.Printf("open log: %v\n", err)
		return
	}
	defer fp.Close()
	log.SetOutput(fp)
	for {
		log.Println("tick from billing")
		time.Sleep(3 * time.Second)
	}
}
```

`O_APPEND` 保证多次启动时接着文件末尾写，`O_CREATE` 在文件不存在时创建。这种写法没有级别，也没有按大小切分，所有调用都进同一个文件。

## 自己做一个带级别的记录器

一个够用的教学实现至少要满足：

- 输出目的地可以换，终端和文件都算 `io.Writer`
- 区分严重程度：debug、trace、info、warning、error、fatal
- 有开关。开发时全部放行，上线后只留下 info 以及更严重的
- 每条带上时间、文件名、行号、函数名、级别和正文
- 文件大了以后要切分（下面的示例先把内容打到标准输出，切分留作下一步）

级别用一组从轻到重的常量表示，数值越大越严重。构造时传入门槛，轻于门槛的调用直接返回。

```go
package main

import (
	"errors"
	"fmt"
	"path"
	"runtime"
	"strings"
	"time"
)

type Severity uint16

const (
	sevUnknown Severity = iota
	sevDebug
	sevTrace
	sevInfo
	sevWarn
	sevError
	sevFatal
)

type Recorder struct {
	floor Severity
}

func parseSeverity(raw string) (Severity, error) {
	switch strings.ToLower(raw) {
	case "debug":
		return sevDebug, nil
	case "trace":
		return sevTrace, nil
	case "info":
		return sevInfo, nil
	case "warning":
		return sevWarn, nil
	case "error":
		return sevError, nil
	case "fatal":
		return sevFatal, nil
	default:
		return sevUnknown, errors.New("unknown severity")
	}
}

func severityName(s Severity) string {
	switch s {
	case sevDebug:
		return "debug"
	case sevTrace:
		return "trace"
	case sevInfo:
		return "info"
	case sevWarn:
		return "warning"
	case sevError:
		return "error"
	case sevFatal:
		return "fatal"
	default:
		return "unknown"
	}
}

func caller(skip int) (fn, file string, line int) {
	pc, full, line, ok := runtime.Caller(skip)
	if !ok {
		return "?", "?", 0
	}
	return runtime.FuncForPC(pc).Name(), path.Base(full), line
}

func NewRecorder(raw string) Recorder {
	floor, err := parseSeverity(raw)
	if err != nil {
		panic(err)
	}
	return Recorder{floor: floor}
}

func (r Recorder) allow(s Severity) bool {
	return s >= r.floor
}

func emit(s Severity, text string) {
	stamp := time.Now().Format("2006-01-02 15:04:05")
	fn, file, line := caller(3)
	fmt.Printf("[%s] [%s] [%s %s:%d] %s\n", stamp, severityName(s), file, fn, line, text)
}

func (r Recorder) Debug(text string) {
	if r.allow(sevDebug) {
		emit(sevDebug, text)
	}
}

func (r Recorder) Trace(text string) {
	if r.allow(sevTrace) {
		emit(sevTrace, text)
	}
}

func (r Recorder) Info(text string) {
	if r.allow(sevInfo) {
		emit(sevInfo, text)
	}
}

func (r Recorder) Warning(text string) {
	if r.allow(sevWarn) {
		emit(sevWarn, text)
	}
}

func (r Recorder) Error(text string) {
	if r.allow(sevError) {
		emit(sevError, text)
	}
}

func (r Recorder) Fatal(text string) {
	if r.allow(sevFatal) {
		emit(sevFatal, text)
	}
}

func main() {
	rec := NewRecorder("error")
	for i := 0; i < 2; i++ {
		rec.Debug("cache miss on sku")
		rec.Info("request accepted")
		rec.Warning("retry scheduled")
		rec.Error("upstream timeout")
		rec.Fatal("ledger corrupted")
		fmt.Println("----")
		time.Sleep(time.Second)
	}
}
```

`caller(3)` 里的 3 表示跳过 `caller`、`emit` 以及 `Debug` 这一层，让行号落在业务调用处。门槛设成 `error` 时，debug、info、warning 都不会打印。把门槛降到 `debug` 则全部出现。

时间格式必须使用 Go 的参考时间 `2006-01-02 15:04:05`，换成别的数字不会按年月日解析。

## 还没做的部分

- `emit` 目前写死到标准输出。把 `Recorder` 加上 `io.Writer` 字段，就能同时支持文件。
- 按大小或日期切分时，要在写入前检查当前文件长度，超过阈值就关闭并换一个新文件名。
- `Fatal` 这里只是级别名，不会结束进程。若希望和标准库一致，写完后应调用 `os.Exit`。
- 多个 goroutine 同时写同一个文件需要加锁，否则行会交织在一起。
