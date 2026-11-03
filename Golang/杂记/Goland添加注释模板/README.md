# Goland添加注释模板

## 前言

从 Java 转到 Go 之后，注释习惯会明显变轻。Java 里常见成块的文档注释，字段、作者、日期各占一行；Go 官方风格更偏向紧挨声明的 `//` 行注释。团队若仍希望新文件和导出函数带一块固定说明，可以在 GoLand 里做两套模板：一套在新建文件时插入，一套在写函数时用缩写展开。

## 类模板定义

打开 Settings → Editor → File and Code Templates，在 Go 相关条目里找到 Go File。把默认内容换成下面这样，保存后每次 New → Go File 都会带上包名和一块说明：

```go
package ${GO_PACKAGE_NAME}

/**
 * 用途：
 * 作者：notebook
 * 日期：${DATE} ${TIME}
 **/
func main() {

}
```

`${GO_PACKAGE_NAME}`、`${DATE}`、`${TIME}` 由 IDE 在创建时替换。建一个名为 `ping.go` 的文件试一次，包名应与所在目录一致，日期应是当天。若模板里写了 `func main`，记得只在真正的入口文件里保留它，库文件应删掉这段函数，避免一个包出现多个入口。

## 方法模板定义

函数注释用 Live Template。仍在 Settings → Editor → Live Templates 中新建一组，缩写设为 `mc`，适用语言勾选 Go，展开内容例如：

```text
/**
 * 说明：$END$
 * 参数：$PARAM$
 * 结果：$RETURN$
 **/
```

触发方式选 Enter。光标先停在说明处，写完后跳到参数和返回值。在某个 Go 函数上方输入 `mc` 再按回车，块注释应立刻出现，且不会插进函数体内部。若缩写没反应，检查模板的上下文是否包含 Go，以及是否与已有缩写重名。
