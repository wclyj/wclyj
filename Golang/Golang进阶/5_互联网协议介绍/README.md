# 互联网协议与 Go 的套接字

程序要跨机器交换数据，就得先知道网络按什么规则组包。这一节从分层讲到套接字，再用 `net` 写出 TCP、处理粘包，以及 UDP。更细的拥塞控制和路由协议需要专门的资料，这里只保留写程序时会直接碰到的部分。

## 协议分层

电脑之间能通信，是因为双方遵守同一组互联网协议。整套协议很大，日常开发接触最多的是下面几层。分层像楼房：上面的一层只依赖紧挨着的下面一层，使用者通常只看见最上面。

不同教材把层次切得不一样，但越往上越靠近用户程序，越往下越靠近网卡和线路。开发时最常用的是五层分法：应用层、传输层、网络层、数据链路层、物理层。

### 物理层

电脑要先用双绞线、光纤或无线电连上网络。物理层规定的是电气和时序特性，任务是把比特送出去。它不解释这些 0 和 1 代表什么。

### 数据链路层

连续的比特没有边界，所以要规定多少比特算一组、每一段什么含义。这是数据链路层的工作。早期各厂商分组方式不同，后来以太网成为局域网里的主流。

以太网把一组信号称为帧，帧分成首部和数据。首部携带发送方、接收方和类型，长度固定 18 字节。数据最短 46 字节，最长 1500 字节。因此一整帧最短 64 字节，最长 1518 字节。更长的内容要拆成多帧。

发送方和接收方用网卡上的 MAC 地址标识。每块网卡出厂时带一个 48 位地址，通常写成 12 个十六进制数字：前半是厂商编号，后半是该厂商的流水号。有了它，帧才能从一块网卡送到另一块。

不知道对方 MAC 时，用 ARP 按 IP 询问。以太网发送时并不精确投递到某一台机器，而是让同一网段里的每台机器都看到帧。各台机器比对首部里的目标 MAC，相同就收下，不同就丢掉。这种发给所有人的方式叫广播。

### 网络层

只靠 MAC 广播有一个硬限制：广播出不了发送者所在的子网，否则互联网上每台机器都要处理所有人的帧，既不现实也没有必要。于是需要另一套地址来判断两台机器是否在同一子网。同一子网走广播，不同子网交给路由器转发。这套地址就是网络层的 IP 地址。

每台主机因此有两种地址。MAC 烧在网卡上，IP 由网络管理员或自动分配协议发放，两者没有固定换算关系。逻辑上先看 IP 决定往哪个子网走，到达目标子网后再用 MAC 找到那块网卡。

目前广泛使用的是 IPv4，地址 32 位，写成四段十进制，范围从 `0.0.0.0` 到 `255.255.255.255`。按 IP 协议封装的数据包也分首部和数据。首部含版本、长度和地址，长 20 到 60 字节，整个数据包最长 65535 字节。

### 传输层

IP 能把数据包送到某一台主机，但主机上同时有浏览器、聊天软件等多个程序。还需要一个 16 位整数标明交给哪个程序，这就是端口，范围 0 到 65535。0 到 1023 多为系统保留，用户程序通常选更大的端口。IP 加端口才能唯一确定网络上的一个程序。

最简单的做法是 UDP：在数据前放上源端口和目标端口。首部只有 8 字节，整个数据报不超过 65535 字节，可以放进一个 IP 包。它实现简单，但发出之后不知道对方是否收到。

TCP 用确认和重传来弥补这个缺口，保证字节流按序到达。代价是状态多、实现复杂、占用更多资源。TCP 报文本身没有固定的应用层长度上限，但为了少在 IP 层再次分片，单个报文段通常不超过 IP 包能容纳的大小。因为 TCP 是字节流而不是一条条消息，接收方看到的是连续字节，于是出现粘包。

### 应用层

传输层交上来的只是字节。互联网是开放的，双方必须事先约定这些字节怎么解释，否则接收方无法还原出发送方的意思。应用层就是这类约定，例如邮件、HTTP 和文件传输。

发送方的 HTTP 内容向下经过每一层时，都会被加上该层的首部。接收方再从外向里逐层剥掉首部，最后得到原来的 HTTP 报文。

## 套接字

套接字最初是 BSD UNIX 的进程通信机制，用来描述 IP 和端口，也是一条通信链路的句柄。它可以理解成 TCP/IP 提供给程序的 API：一组函数，用来在网络上建立连接、收发字节。应用程序通过它发起请求或应答。

底层的分片、重传和路由被挡在这组 API 后面。从设计上看，套接字像一层门面：调用方只按函数约定提交数据，由它组织成对应协议的报文。

## 用 Go 写 TCP

TCP 是面向连接的可靠字节流。连接建立之后，数据像水流一样连续到达，消息之间没有天然边界，所以会粘包。

### 服务端

一个监听端口可以同时接入很多客户端。Go 里每接受一条连接就启动一个 goroutine 去读它，互不阻塞。

处理步骤是：监听端口，接受连接，把连接交给单独的 goroutine。

```go
func serve(conn net.Conn) {
	defer conn.Close()
	buf := make([]byte, 128)
	for {
		n, err := conn.Read(buf)
		if err != nil {
			fmt.Println("read:", err)
			return
		}
		fmt.Printf("%s says %s\n", conn.RemoteAddr(), buf[:n])
	}
}

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:21000")
	if err != nil {
		fmt.Println("listen:", err)
		return
	}
	defer ln.Close()
	for {
		conn, err := ln.Accept()
		if err != nil {
			fmt.Println("accept:", err)
			continue
		}
		go serve(conn)
	}
}
```

`Read` 返回的字节数可能小于缓冲区，也可能把多次 `Write` 拼在一起。不要假设一次 `Read` 正好是一条消息。

### 客户端

```go
func readLine() string {
	in := bufio.NewReader(os.Stdin)
	text, err := in.ReadString('\n')
	if err != nil {
		return ""
	}
	return strings.TrimSpace(text)
}

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:21000")
	if err != nil {
		fmt.Println("dial:", err)
		return
	}
	defer conn.Close()
	_, err = conn.Write([]byte(readLine()))
	if err != nil {
		fmt.Println("write:", err)
	}
}
```

`Dial` 成功只说明三次握手完成。对端关闭后，下一次 `Read` 会得到 `io.EOF`。

## 粘包

### 先看现象

服务端按缓冲区原样打印：

```go
func serve(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	buf := make([]byte, 1024)
	for {
		n, err := reader.Read(buf)
		if err == io.EOF {
			return
		}
		if err != nil {
			fmt.Println("read:", err)
			return
		}
		fmt.Println("got:", string(buf[:n]))
	}
}

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:31000")
	if err != nil {
		fmt.Println("listen:", err)
		return
	}
	defer ln.Close()
	for {
		conn, err := ln.Accept()
		if err != nil {
			fmt.Println("accept:", err)
			continue
		}
		go serve(conn)
	}
}
```

客户端紧凑地写 20 次同一句：

```go
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:31000")
	if err != nil {
		fmt.Println("dial:", err)
		return
	}
	defer conn.Close()
	for i := 0; i < 20; i++ {
		conn.Write([]byte("ping-from-client"))
	}
}
```

服务端往往不会整齐地打印 20 行。一次 `Read` 里可能粘着好几句，例如：

```text
got: ping-from-clientping-from-clientping-from-client
got: ping-from-clientping-from-client
```

发送了 20 次，接收却变成几团字节。这就是粘包，也可能反过来出现半包：一次 `Read` 只拿到一条消息的前半段。

### 原因

TCP 是流，长连接上可以连续写、连续读，协议本身不保留“第几次 Write”这个边界。

粘在一起的情况来自两端：

1. 发送端的 Nagle 算法会等一小会儿，看这段时间里还有没有后续数据，有的话合并后再发出，用来减少小包数量。
2. 接收端若没有及时把套接字缓冲区读空，几次到达的数据会堆在一起，下一次 `Read` 就一次拿回去了。

### 用长度头把边界加回去

接收方必须知道一条消息有多长。做法是封包：在正文前加固定长度的首部，首部里写下正文的字节数。需要丢弃非法包时还可以再加尾部校验。首部长度固定，读出里面的正文长度，就能恰好切出一条消息。

这里约定前 4 个字节是小端序的 `int32` 长度，后面紧跟正文。

```go
package frame

import (
	"encoding/binary"
	"fmt"
	"io"
)

func Wrap(body string) ([]byte, error) {
	if len(body) > int(^uint32(0)>>1) {
		return nil, fmt.Errorf("body too long")
	}
	buf := make([]byte, 4+len(body))
	binary.LittleEndian.PutUint32(buf[:4], uint32(len(body)))
	copy(buf[4:], body)
	return buf, nil
}

func Unwrap(r io.Reader) (string, error) {
	head := make([]byte, 4)
	if _, err := io.ReadFull(r, head); err != nil {
		return "", err
	}
	n := binary.LittleEndian.Uint32(head)
	body := make([]byte, n)
	if _, err := io.ReadFull(r, body); err != nil {
		return "", err
	}
	return string(body), nil
}
```

`ReadFull` 在字节不够时继续读，因此半包会被留到下一次系统调用，而不是被当成一条残缺消息。长度来自对端，正式环境要设上限，避免对端声称一个巨大的长度把内存撑满。

字节序是多字节整数在内存或线路上的排列方式。小端把低位字节放在前面，大端把高位字节放在前面。双方必须使用同一种，否则长度会被解释成另一个数。网络协议传统上多用大端，本例为了和常见教学代码对照，固定用小端，关键是两端一致。

服务端改为按帧读取：

```go
func serve(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	for {
		msg, err := frame.Unwrap(reader)
		if err == io.EOF {
			return
		}
		if err != nil {
			fmt.Println("unwrap:", err)
			return
		}
		fmt.Println("got:", msg)
	}
}
```

客户端每次先封包再写：

```go
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:31000")
	if err != nil {
		fmt.Println("dial:", err)
		return
	}
	defer conn.Close()
	for i := 0; i < 20; i++ {
		pkt, err := frame.Wrap("ping-from-client")
		if err != nil {
			fmt.Println("wrap:", err)
			return
		}
		if _, err = conn.Write(pkt); err != nil {
			fmt.Println("write:", err)
			return
		}
	}
}
```

这时服务端会按 20 条独立消息打印。`Write` 本身也不保证一次写完全部字节，严谨的发送端应使用 `io.WriteFull` 之类的循环，直到全部写出或出错。

## 用 Go 写 UDP

UDP 是无连接的传输协议，不握手就能发送。它不保证到达、不保证顺序、也不去重，但头部开销小、延迟低，常用于视频直播、探测和日志这类可以容忍丢失的流量。

### 服务端

```go
func main() {
	conn, err := net.ListenUDP("udp", &net.UDPAddr{
		IP:   net.IPv4(0, 0, 0, 0),
		Port: 32000,
	})
	if err != nil {
		fmt.Println("listen:", err)
		return
	}
	defer conn.Close()
	buf := make([]byte, 1024)
	for {
		n, addr, err := conn.ReadFromUDP(buf)
		if err != nil {
			fmt.Println("read:", err)
			continue
		}
		fmt.Printf("from %s %q (%d bytes)\n", addr, buf[:n], n)
		if _, err = conn.WriteToUDP(buf[:n], addr); err != nil {
			fmt.Println("write:", err)
		}
	}
}
```

每个数据报都带着对端地址，回复时要写回这个地址。一次 `ReadFromUDP` 对应一个数据报，不会把两个数据报粘进同一次读取，这是它和 TCP 流的差别。数据报仍然可能丢失或乱序。

### 客户端

```go
func main() {
	conn, err := net.DialUDP("udp", nil, &net.UDPAddr{
		IP:   net.IPv4(127, 0, 0, 1),
		Port: 32000,
	})
	if err != nil {
		fmt.Println("dial:", err)
		return
	}
	defer conn.Close()
	if _, err = conn.Write([]byte("hello-udp")); err != nil {
		fmt.Println("write:", err)
		return
	}
	buf := make([]byte, 4096)
	n, peer, err := conn.ReadFromUDP(buf)
	if err != nil {
		fmt.Println("read:", err)
		return
	}
	fmt.Printf("echo %q from %s (%d bytes)\n", buf[:n], peer, n)
}
```

`DialUDP` 只是把目标地址记在本地套接字上，没有握手。对方没启动时，`Write` 仍可能成功，错误要到后续读取或 ICMP 反馈时才暴露。

## 选型时的提醒

- 消息型协议走 TCP 时一定要有帧格式，不能靠 `sleep` 碰运气分开两条消息。
- UDP 少了粘包，却多了丢包和乱序，应用层若需要可靠，就得自己做确认和序号。
- 服务端每个连接一个 goroutine 很方便，但要给 `Read` 设置截止时间，否则对端不说话时 goroutine 会一直占着。
