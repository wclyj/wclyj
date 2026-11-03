# Go 中的日期函数

## time包

业务里几乎总会碰到“现在几点”和“两个时刻差多久”。标准库 `time` 同时负责墙上时钟的展示，以及用单调时钟做间隔测量。

## time.Now获取当前时间

```go
cur := time.Now()
fmt.Printf("%d-%02d-%02d\n", cur.Year(), int(cur.Month()), cur.Day())
```

`Month()` 的底层是整数枚举，打印前转成 `int`，`%02d` 才能把 3 月补成 `03`。

## 格式化日期

`Time` 自带 `Format`。布局串不是 `Y-m-d`，而是 Go 的诞生时刻：2006 年 1 月 2 日 15 点 04 分 05 秒。记数字 `2006 1 2 3 4 5` 即可。

```go
wall := time.Now()
fmt.Println(wall.Format("2006-01-02 15:04:05")) // 24 小时
fmt.Println(wall.Format("2006-01-02 03:04:05 PM"))
```

`15` 表示 24 小时制的小时，`03` 表示 12 小时制。月份、日期、分、秒都要用对应的参考数字，不能换成别的数字还指望它代表同一字段。

## 获取当前时间戳

Unix 时间戳是从 1970-01-01 00:00:00 UTC 起累计的秒数，不是毫秒。`Unix()` 返回秒，`UnixMilli()` 返回毫秒，`UnixNano()` 返回纳秒。

```go
snap := time.Now()
sec := snap.Unix()
nano := snap.UnixNano()
fmt.Println(sec, nano)
```

## 时间戳转日期字符串

`time.Unix` 的两个参数分别是秒和纳秒偏移，不是毫秒。

```go
when := time.Unix(1_596_000_000, 0)
fmt.Println(when.Format("2006-01-02 15:04:05"))
```

## 日期字符串转换成时间戳

没有时区信息的字符串要用 `ParseInLocation`，否则按 UTC 解析，和本地挂钟会差几个小时。

```go
layout := "2006-01-02 15:04:05"
text := "2024-05-01 09:30:00"
parsed, err := time.ParseInLocation(layout, text, time.Local)
if err != nil {
    panic(err)
}
fmt.Println(parsed.Unix())
```

## 时间间隔

`time.Duration` 用纳秒表示两个时刻之间的跨度，底层是 `int64`，大约能覆盖 290 年。常用常量：

| 常量 | 含义 |
| --- | --- |
| `Nanosecond` | 1 纳秒 |
| `Microsecond` | 10^3 纳秒 |
| `Millisecond` | 10^6 纳秒 |
| `Second` | 10^9 纳秒 |
| `Minute` | 60 秒 |
| `Hour` | 60 分钟 |

## 时间操作函数

把一个 `Duration` 加到时刻上，用 `Add`：

```go
func (t Time) Add(d Duration) Time
```

```go
start := time.Now()
next := start.Add(2 * time.Hour)
fmt.Println(next)
```

相邻的方法还有：`Sub` 得到两个时刻的差，`Equal` 判断时刻是否相同，`Before` / `After` 比较先后。`AddDate` 则按年、月、日做日历运算，能正确处理月末。

## 定时器

方式 1：`time.NewTicker` 按固定间隔往通道送时刻，用完要 `Stop`，避免泄漏。

```go
tick := time.NewTicker(time.Second)
defer tick.Stop()
hits := 0
for ts := range tick.C {
    fmt.Println(ts.Format("15:04:05"))
    hits++
    if hits == 3 {
        return
    }
}
```

方式 2：`time.Sleep` 让当前协程让出一段时间，适合简单的间隔循环。

```go
for step := 0; step < 3; step++ {
    time.Sleep(time.Second)
    fmt.Println("过了一秒")
}
```

`Sleep` 不提供取消通道；若要能提前停，优先用 `Ticker` 或 `time.After` 配合 `select`。
