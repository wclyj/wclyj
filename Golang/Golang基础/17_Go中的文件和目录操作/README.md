# Go 中的文件和目录操作

## 文件的读取

### 通过os.Open方法读取文件

`os.Open` 只读打开。先判断错误再 `defer Close`。只拼接本次读到的前 `n` 个字节。

```go
f, err := os.Open("notes/demo.txt")
if err != nil { return }
defer f.Close()
buf, all := make([]byte, 64), []byte{}
for {
    n, err := f.Read(buf)
    all = append(all, buf[:n]...)
    if err == io.EOF { break }
}
fmt.Println(string(all))
```

### 通过bufio的方式读取

按行读时分隔符用 `'\n'`。遇到 EOF 时最后一段可能没有换行，退出前仍要收下。

```go
f, _ := os.Open("notes/demo.txt")
defer f.Close()
rd := bufio.NewReader(f)
var text string
for {
    line, err := rd.ReadString('\n')
    text += line
    if err == io.EOF { break }
}
fmt.Println(text)
```

### 通过ioutil读取

小文件一次读完。1.16 起用 `os.ReadFile`，`ioutil.ReadFile` 是旧名。

```go
raw, err := os.ReadFile("notes/demo.txt")
fmt.Println(err, string(raw))
```

## 文件的写入

写入前用 `os.OpenFile`。三个参数分别是路径、打开标志、权限。

- name：目标路径
- flag：可以用按位或组合
  - `os.O_RDONLY`：只读
  - `os.O_WRONLY`：只写
  - `os.O_RDWR`：读写
  - `os.O_CREATE`：不存在就创建
  - `os.O_TRUNC`：打开时截断
  - `os.O_APPEND`：写到末尾
- perm：Unix 权限八进制。读是 `04`，写是 `02`，执行是 `01`，`0644` 表示所有者可读写、其他人只读

### 通过OpenFile打开文件写入

```go
f, err := os.OpenFile("notes/demo.txt", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
if err != nil {
    fmt.Println(err)
    return
}
defer f.Close()
_, _ = f.WriteString("追加一行\n")
```

### 通过bufio写入

`bufio.Writer` 先写内存，必须 `Flush` 才会进磁盘。

```go
f, err := os.OpenFile("notes/demo.txt", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
if err != nil {
    fmt.Println(err)
    return
}
defer f.Close()

bw := bufio.NewWriter(f)
_, _ = bw.WriteString("经缓冲写入\n")
if err = bw.Flush(); err != nil {
    fmt.Println(err)
}
```

## 通过ioutil写入

整段覆盖用 `os.WriteFile`（旧名 `ioutil.WriteFile`），不存在会创建，存在会截断。

```go
err := os.WriteFile("notes/demo.txt", []byte("hello file"), 0644)
if err != nil {
    fmt.Println(err)
}
```

## 文件复制

小文件读进内存再写出。大文件改用 `io.Copy`。

```go
raw, err := os.ReadFile("notes/demo.txt")
if err != nil {
    fmt.Println("读失败", err)
    return
}
if err = os.WriteFile("notes/demo-copy.txt", raw, 0644); err != nil {
    fmt.Println("写失败", err)
}
```

## 创建目录

`Mkdir` 只建一层。父目录也要一起建时用 `MkdirAll`。

```go
if err := os.MkdirAll("data/inbox", 0755); err != nil {
    fmt.Println(err)
}
```

## 删除操作

`Remove` 删除文件或空目录。目录非空时用 `RemoveAll`。

```go
_ = os.Remove("data/inbox/a.txt")
_ = os.Remove("data/empty")
_ = os.RemoveAll("data/inbox")
```

## 重命名

`Rename` 接收旧路径和新路径。同一文件系统上通常只改目录项。

```go
if err := os.Rename("notes/demo-copy.txt", "notes/demo-bak.txt"); err != nil {
    fmt.Println(err)
}
```
