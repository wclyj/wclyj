# decimal

在 Go 里做定点小数运算的库。它用整数保存有效数字，另记小数位数，因此可以精确表示 `0.1` 这种二进制浮点无法落地的数。小数点后的位数上限大约是 2^31，日常的金额和税率远达不到这个边界。

## Features

- 零值就是 0，不必先调用构造函数也能参与运算
- 加、减、乘不丢失精度
- 除法由调用方指定保留的小数位
- 实现了 `database/sql` 需要的扫描与写入接口，能直接进出数据库
- 能序列化成 JSON 和 XML，也能从这两种格式读回来

## Install

在模块目录执行：

```bash
go get github.com/shopspring/decimal
```

## Requirements

工具链需要 Go 1.7 或更高。更老的版本缺少本库依赖的语言特性。

## Usage

下面用单价、数量、费率和税率算一笔订单。所有小数都从字符串构造，避免先掉进 `float64`：

```go
package main

import (
	"fmt"

	"github.com/shopspring/decimal"
)

func main() {
	unit, err := decimal.NewFromString("86.50")
	if err != nil {
		panic(err)
	}
	qty := decimal.NewFromInt(4)
	fee, _ := decimal.NewFromString("0.02")
	tax, _ := decimal.NewFromString("0.06")

	sub := unit.Mul(qty)
	withFee := sub.Mul(fee.Add(decimal.NewFromInt(1)))
	grand := withFee.Mul(tax.Add(decimal.NewFromInt(1)))

	fmt.Println("sub:", sub)
	fmt.Println("with fee:", withFee)
	fmt.Println("tax:", grand.Sub(withFee))
	fmt.Println("grand:", grand)
	fmt.Println("rate:", grand.Sub(withFee).Div(withFee))
}
```

`Mul`、`Add`、`Sub`、`Div` 都返回新的 `Decimal`，原来的值不变。除法要留意除数不能为 0，并在需要固定小数位时改用带精度参数的除法，而不是依赖默认位数。

## Documentation

完整的方法列表以模块文档为准，重点看构造函数（字符串、整数、大整数）以及 `Round`、`Cmp`、`StringFixed`。比较大小用 `Cmp` 或 `Equal`，不要把小数先转成 `float64` 再比较。

## Production Usage

这个库被用在计费和订单金额上。生产环境引入前，先约定全项目统一的舍入方式：是银行家舍入还是一律进位，小数保留几位。混用 `Round` 和 `RoundBank` 会让同一笔账在不同服务里差一分钱。

## FAQ

#### Why don't you just use float64?

`float64` 以及任何二进制浮点都无法精确表示 `0.1`。把十个 `0.1` 相加，打印出来常常不是 `1`，而是一长串 9。误差单独看很小，累加到对账时就会和上下游差出几分。金额、税率、库存单价都不应走这条路。

#### Why don't you just use big.Rat?

`big.Rat` 适合保存有理数，分子分母都精确。拿它表示钱却不方便：三分之一打印成三位小数时是 `0.333`，三个 `0.333` 加起来只有 `0.999`，剩下的 `0.001` 在字符串层面消失了。定点小数在设定精度后，每个打印出来的数字就等于内部的数，拆分后的余数会落在某一份上，账能对平。

把一笔钱分成三份时，不能三个人都拿 `N/3`。应让两个人拿按精度截断后的商，最后一个人拿 `N` 减去前两份，余数才不会凭空消失。`Decimal` 把这个余数留在十进制上，比在分数之间来回约分更容易核对。

#### Why isn't the API similar to big.Int's?

`big.Int` 为了少分配内存，很多方法会改接收者，例如 `z := new(big.Int).Add(x, y)`。不熟悉的人写成 `a.Add(a, b)` 时，既改了 `a`，又让返回值和 `a` 成为同一块内存，别的别名也会被悄悄改掉。

`Decimal` 的方法一律返回新值，不修改原对象。`a = b` 仍然只是复制结构体里的系数引用语义要按值理解：运算不会把旧的 `a` 改掉。代价是每次运算可能多一次分配。选择这个库，通常是因为正确性比那点分配更重要；热点路径上的纯整数计数仍应使用原生整数。

再看舍入。金额通常保留两位，乘法之后先比大小，再按约定收成两位，避免把一长串小数直接展示给用户：

```go
a := decimal.RequireFromString("10.015")
b := decimal.RequireFromString("10.025")
fmt.Println(a.Round(2), b.Round(2))
fmt.Println(a.Cmp(b)) // 小于则 -1
```

`Round` 与银行家舍入的差异要在项目里定死。同一笔 1.005 在不同舍入规则下可能变成 1.00 或 1.01。入库前用 `StringFixed(2)` 得到稳定文本，读出时再用 `NewFromString`，不要经过 `float64`。

和数据库配合时，列用 `DECIMAL(18,4)` 这类定点类型，模型字段用 `decimal.Decimal`。扫描接口会把驱动返回的文本或字节还原成系数和小数位。若列是 `FLOAT`，精度在进库前就已经丢了，库也救不回来。

零值可以直接 `Add`。未初始化的结构体表示 0，适合作为累加器的起点：

```go
var sum decimal.Decimal
for _, text := range []string{"1.20", "0.30", "4.50"} {
    sum = sum.Add(decimal.RequireFromString(text))
}
fmt.Println(sum)
```

除法要单独处理。`Div` 使用库默认的精度，循环小数会被截断。展示税率、分成时改用带精度的除法，并检查除数是否为零，否则会 panic。分成三份的余数留给最后一份，前两份用同一精度截断，三份相加才能回到原数。

## License

以 MIT 许可证发布。实现上借鉴过早期的定点小数库，并在 API 和数据库支持上做了较大改动。使用时保留版权声明即可。
