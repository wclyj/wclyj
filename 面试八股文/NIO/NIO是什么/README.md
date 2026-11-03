# Java NIO

## 概念

Java NIO 从 1.4 开始提供，常被说成新输入输出，也常被说成非阻塞输入输出。它和原来的流式输入输出要解决的问题一样，都是把字节搬进搬出，但使用方式不同：数据先放进缓冲区，再通过通道搬运，需要同时盯住很多连接时再加选择器。文件读写可以少几次用户态和内核态之间的拷贝。

名字里的 N 最初强调的是新接口。非阻塞是通道可以切换的一种模式，不是所有 NIO 程序都自动非阻塞。`FileChannel` 就不能切成非阻塞。

## IO和NIO的区别

| 经典 IO | NIO |
| --- | --- |
| 面向流 | 面向缓冲区 |
| 读写会堵住当前线程 | 通道可以改成不堵住 |
| 没有多路复用 | 用选择器同时盯住多条通道 |

经典流是单向的。读一个文件要开输入流，写出去要再开输出流，数据在一条流里只能朝一个方向走。

NIO 的缓冲区没有这个方向限制。同一块缓冲可以先写进数据，翻转之后再读出来，通道也可以按打开方式同时承担读和写。

通道可以比作站台之间的轨道，它只负责把两端连上。真正装货的是缓冲区。没有缓冲，通道不知道这次要搬哪些字节。

## 通道和缓冲区

这两样是 NIO 的核心。通道表示一条已经打开的连接，对端可能是文件，也可能是套接字。要用这套接口，就先拿到通道，再准备一块放数据的缓冲，然后对缓冲做读、写和翻转。

一句话：通道负责搬，缓冲负责装。

## 缓冲区 Buffer

缓冲是按类型分好的数组，外加几个指针。除了布尔，每种基本类型都有对应实现。

- `ByteBuffer`：最常用，文件和网络最终都是字节
- `CharBuffer`
- `ShortBuffer`
- `IntBuffer`
- `LongBuffer`
- `FloatBuffer`
- `DoubleBuffer`

### 缓冲区中的方法

各种缓冲的指针语义一致，用 `allocate` 在堆里划出一块。

和读写直接相关的方法：

- `put`：往当前位置写入
- `get`：从当前位置读出
- `hasRemaining`：当前位置到上限之间是否还有剩余
- `remaining`：还剩多少个元素
- `mark`：记下现在的位置
- `reset`：把位置退回刚才的记号
- `flip`：从写切换到读
- `rewind`：再读一遍已有内容
- `clear`：把指针复位，准备重新写

### 缓冲区中的核心属性

四个指针决定哪一段才是有效数据。

- `capacity`：容量，创建之后不能再变
- `limit`：上限，上限之后的位置不允许读写
- `position`：下一个要读或要写的下标
- `mark`：随意记下的位置，供 `reset` 使用

它们始终满足：`0 <= mark <= position <= limit <= capacity`。没调用过 `mark` 时，记号处于未设置状态。

### 相关操作

先划一块 32 字节的缓冲，看三个指针的初值。容量 32 只是为了把数字写短，真实拷贝常用 8KB 一类的大小。

```java
ByteBuffer buf = ByteBuffer.allocate(32);
System.out.println("刚分配 position=" + buf.position()
        + " limit=" + buf.limit()
        + " capacity=" + buf.capacity());
```

此时位置是 0，上限和容量都是 32。接着写入 6 个字节。

```java
byte[] raw = "港区A1".getBytes(java.nio.charset.StandardCharsets.UTF_8);
buf.put(raw);
System.out.println("写入后 position=" + buf.position()
        + " limit=" + buf.limit()
        + " capacity=" + buf.capacity());
```

“港区A1”的 UTF-8 长度是 8：两个汉字各 3 字节，加上 `A` 和 `1`。位置变成 8，上限仍是 32。如果立刻 `get`，会从下标 8 往后读，读到的是还没写过的零，而不是刚才的文字。所以读之前要翻转。

```java
buf.flip();
System.out.println("翻转后 position=" + buf.position()
        + " limit=" + buf.limit()
        + " capacity=" + buf.capacity());
byte[] dst = new byte[buf.remaining()];
buf.get(dst);
System.out.println(new String(dst, java.nio.charset.StandardCharsets.UTF_8));
```

翻转把上限收到原来的位置，再把位置归零。于是可读区间正好是这 8 个字节。读完后位置追上上限。

指针变化可以记成四步：

1. 分配后：`position=0`，`limit=capacity`
2. 写入 n 个字节：`position=n`，`limit` 不动
3. 翻转：`limit=n`，`position=0`
4. 读完：`position=limit=n`

完整例子把重复读和清空也走一遍。

```java
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;

public class BufferPointerLab {
    public static void main(String[] args) {
        ByteBuffer buf = ByteBuffer.allocate(32);
        print("分配", buf);

        buf.put("港区A1".getBytes(StandardCharsets.UTF_8));
        print("写入", buf);

        buf.flip();
        print("翻转", buf);

        byte[] dst = new byte[buf.remaining()];
        buf.get(dst);
        System.out.println(new String(dst, StandardCharsets.UTF_8));
        print("读完", buf);

        buf.rewind();
        print("重读", buf);

        buf.clear();
        print("清空指针", buf);
        System.out.println("首字节仍在:" + buf.get(0));
    }

    static void print(String stage, ByteBuffer buf) {
        System.out.println(stage + " position=" + buf.position()
                + " limit=" + buf.limit()
                + " capacity=" + buf.capacity());
    }
}
```

输出里，分配阶段三个数是 0、32、32；写入后位置是 8；翻转后位置回到 0、上限变成 8；读完位置是 8。`clear` 只把位置改回 0、上限改回容量，数组里的字节还在，所以按下标仍能看到首字节。这些字节处于“被遗忘”的状态，下一次 `put` 会从位置 0 覆盖它们。若写的比上一次短，尾部会留下旧数据，所以不能在 `clear` 之后不看 `position` 就把整个数组当新内容。

`mark` 和 `reset` 适合读到一半先做个记号。例如先读出类型码，发现不是自己要的包，再退回记号重新按另一种格式解析。

### 堆内缓冲与堆外缓冲

堆内缓冲用 `allocate` 创建，内存算在虚拟机堆上。经典流和这种缓冲在碰到真正的系统调用时，往往还要把字节抄到一块堆外中转区，多一次复制。

堆外缓冲用 `allocateDirect` 创建，内存向操作系统申请。原生读写可以直接用这块地址，少一次从堆到中转区的复制。文件映射是另一种堆外缓冲：`FileChannel.map` 返回 `MappedByteBuffer`，文件的一段区间直接映到地址空间。

堆外内存不由垃圾回收器搬移，回收时机也更晚。缓冲对象本身被回收时才会释放底层内存，所以短命的小缓冲不适合频繁直接分配。

更细一点：

- 字节缓冲要么在堆内，要么在堆外。堆外时，虚拟机会尽量让一次原生调用直接使用这块内存，避免先抄到临时数组
- `allocateDirect` 的申请和释放都比堆内贵。它适合那些活得久、又反复参与大块读写的缓冲，而不是每处理一行就申请一次
- 映射缓冲依赖文件区间。区间如果后来不可访问，读取可能抛出不确定的异常，内容也不保证被改写
- 调用 `isDirect()` 就能区分两种缓冲。性能敏感的代码可以据此决定要不要走特殊路径

堆外缓冲还要小心大小。它不受堆上限约束，配得过大时，进程会先把机器内存吃满，而堆监控看起来仍很空。

## 通道 Channel

### 概念

通道由 `java.nio.channels` 定义，表示源头和目标之间已经打开的连接。它和旧流一样负责传输，但不能自己存放字节，必须搭配缓冲。

早期由处理器直接响应每一次读写，设备一慢，处理器就空转。后来增加了直接内存访问：设备把块写进内存，只在开始和结束时打扰处理器。连接一多，这种专用通路也不够用，于是把搬运交给可以独立调度的通道来做。对编写业务的人来说，通道和流的差别主要在接口：通道面向缓冲，并且部分实现可以非阻塞、可以多路复用。

### 相关实现类

通道连接两个端点，自己不留数据。

常见实现：

- `FileChannel`：文件
- `SocketChannel`：面向连接的客户端套接字
- `ServerSocketChannel`：负责接收连接
- `DatagramChannel`：报文套接字

`Channels` 这个工具类还能把传统流包成通道，但它本身不是上面四种端点。

### 获取通道

支持通道的旧类提供 `getChannel`。

本地文件：

- `FileInputStream`
- `FileOutputStream`
- `RandomAccessFile`

网络：

- `Socket`
- `ServerSocket`
- `DatagramSocket`

1.7 起，各个通道可以用静态的 `open` 直接打开，并用 `StandardOpenOption` 声明读、写、创建等意图。`Files.newByteChannel` 则从路径得到一条可寻址的字节通道。

### 利用通道完成文件的复制

堆内缓冲适合边读边写，缓冲可以很小，文件多大都能循环搬完。

```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class HeapCopyLab {
    public static void main(String[] args) throws Exception {
        try (FileInputStream in = new FileInputStream("/tmp/nio-lab/manifest.txt");
             FileOutputStream out = new FileOutputStream("/tmp/nio-lab/manifest-copy.txt");
             FileChannel source = in.getChannel();
             FileChannel target = out.getChannel()) {
            ByteBuffer buf = ByteBuffer.allocate(16);
            while (source.read(buf) != -1) {
                buf.flip();
                target.write(buf);
                buf.clear();
            }
        }
    }
}
```

每次 `read` 把字节放进缓冲，翻转后才能写到目标通道。写完必须 `clear`，否则下一次读会从上次的位置继续，很快就没有剩余空间，循环看起来还在转，数据却不再前进。`read` 返回 -1 表示源已经结束。这里用 try-with-resources 关闭流，通道会跟着关掉，不必在 `finally` 里逐个判空。

文件映射适合源和目标都可以放进地址空间的场景。它少了用户态循环，但映射大文件会占用虚存，并且 `CREATE_NEW` 在目标已存在时会失败。

```java
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

public class MappedCopyLab {
    public static void main(String[] args) throws Exception {
        try (FileChannel source = FileChannel.open(
                     Paths.get("/tmp/nio-lab/manifest.txt"), StandardOpenOption.READ);
             FileChannel target = FileChannel.open(
                     Paths.get("/tmp/nio-lab/manifest-mapped.txt"),
                     StandardOpenOption.READ, StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
            MappedByteBuffer inMap = source.map(FileChannel.MapMode.READ_ONLY, 0, source.size());
            MappedByteBuffer outMap = target.map(FileChannel.MapMode.READ_WRITE, 0, source.size());
            outMap.put(inMap);
        }
    }
}
```

两条通道之间还可以直接交接，由系统决定是走内核拷贝还是更省的传输。业务代码不再自己管翻转。

```java
import java.nio.channels.FileChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

public class TransferCopyLab {
    public static void main(String[] args) throws Exception {
        try (FileChannel source = FileChannel.open(
                     Paths.get("/tmp/nio-lab/manifest.txt"), StandardOpenOption.READ);
             FileChannel target = FileChannel.open(
                     Paths.get("/tmp/nio-lab/manifest-transfer.txt"),
                     StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
            long moved = 0;
            while (moved < source.size()) {
                moved += source.transferTo(moved, source.size() - moved, target);
            }
        }
    }
}
```

`transferTo` 不保证一次搬完，所以要用返回值累加。目标用 `CREATE` 而不是只在不存在时创建，重复运行笔记里的例子才不会因为文件已在而失败。

### 分散读取与聚集写入

分散读取把通道里连续的字节按顺序填进多块缓冲。第一块写满自己的剩余空间后，才轮到下一块。

聚集写入则按数组顺序，把每块缓冲里位置到上限之间的字节依次送进通道。

```java
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.StandardCharsets;

public class ScatterGatherLab {
    public static void main(String[] args) throws Exception {
        try (RandomAccessFile file = new RandomAccessFile("/tmp/nio-lab/scatter.txt", "rw");
             FileChannel channel = file.getChannel()) {
            ByteBuffer head = ByteBuffer.allocate(4);
            ByteBuffer body = ByteBuffer.allocate(32);
            channel.read(new ByteBuffer[]{head, body});
            head.flip();
            body.flip();
            System.out.println(StandardCharsets.UTF_8.decode(head));
            System.out.println(StandardCharsets.UTF_8.decode(body));

            head.rewind();
            body.rewind();
            try (RandomAccessFile out = new RandomAccessFile("/tmp/nio-lab/gather.txt", "rw");
                 FileChannel target = out.getChannel()) {
                target.write(new ByteBuffer[]{head, body});
            }
        }
    }
}
```

若文件以 `GATE港口开放` 开头，4 字节的头会拿走 `GATE`，剩下的汉字落在第二块。缓冲数组的顺序就是数据被切开的顺序，不是按某个分隔符切。聚集写回去时，也是按这个顺序拼接。某块缓冲忘记翻转的话，写出去的会是位置之后的空字节。

### 字符集

编码把字符变成字节，解码把字节还原成字符。两端的字符集必须一致，否则汉字会变成替换符或直接抛出畸形输入。

```java
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.charset.Charset;
import java.nio.charset.CharsetDecoder;
import java.nio.charset.CharsetEncoder;

public class CharsetLab {
    public static void main(String[] args) throws Exception {
        Charset gbk = Charset.forName("GBK");
        CharsetEncoder encoder = gbk.newEncoder();
        CharsetDecoder decoder = gbk.newDecoder();

        CharBuffer chars = CharBuffer.allocate(16);
        chars.put("夜航顺利");
        chars.flip();

        ByteBuffer bytes = encoder.encode(chars);
        while (bytes.hasRemaining()) {
            System.out.printf("%02x ", bytes.get());
        }
        System.out.println();

        bytes.flip();
        CharBuffer back = decoder.decode(bytes);
        System.out.println(back);
    }
}
```

“夜航顺利”四个汉字在 GBK 下是 8 个字节。用 UTF-8 的解码器去解这 8 个字节会对不上。缓冲版的编码比 `String.getBytes` 更适合已经握着 `CharBuffer` 的通道代码。编码之后位置在末尾，要再翻转一次才能解码。

## NIO的非阻塞式网络通信

经典阻塞调用必须等连接到来、或者等对方把字节送满，线程才能继续。一个空闲客户端就能占住一条线程。

选择器允许很多通道登记到同一个复用器上。只有某条通道真正可读、可写或可接收时，选择器才返回。服务端于是用很少的线程照看大量连接。通道要先改成非阻塞，才能注册进去。

## 使用NIO完成网络通信的三个核心

- 通道负责连接。可选中的那一支下面有 `SocketChannel`、负责接入的 `ServerSocketChannel`，以及面向报文的 `DatagramChannel`。管道两端的 `Pipe.SinkChannel` 和 `Pipe.SourceChannel` 也是通道，但不能注册到选择器
- 缓冲负责装字节
- 选择器是多路复用器，用来观察哪些可选通道已经就绪

三者缺一不可：没有通道就没有连接，没有缓冲就没地方放字节，没有选择器就退回到“一条连接一条线程”或者自己空转轮询。

## 使用阻塞式IO完成网络通信

下面先写一个会堵住的文件接收端。它用通道，但没有切非阻塞，`accept` 和 `read` 都会等待。

```java
public static void serve() throws java.io.IOException {
    java.nio.file.Path dest = java.nio.file.Paths.get("/tmp/nio-lab/inbox.bin");
    try (java.nio.channels.ServerSocketChannel listener = java.nio.channels.ServerSocketChannel.open();
         java.nio.channels.FileChannel file = java.nio.channels.FileChannel.open(
                 dest, java.nio.file.StandardOpenOption.WRITE, java.nio.file.StandardOpenOption.CREATE)) {
        listener.bind(new java.net.InetSocketAddress(19090));
        java.nio.channels.SocketChannel client = listener.accept();
        java.nio.ByteBuffer buf = java.nio.ByteBuffer.allocate(64);
        while (client.read(buf) != -1) {
            buf.flip();
            file.write(buf);
            buf.clear();
        }
        client.close();
    }
}
```

客户端把本地文件推上去。写完之后要关掉输出，对方的 `read` 才能返回 -1，否则服务端会一直等下一段。

```java
public static void send() throws java.io.IOException {
    try (java.nio.channels.SocketChannel socket = java.nio.channels.SocketChannel.open(
                 new java.net.InetSocketAddress("127.0.0.1", 19090));
         java.nio.channels.FileChannel file = java.nio.channels.FileChannel.open(
                 java.nio.file.Paths.get("/tmp/nio-lab/manifest.txt"), java.nio.file.StandardOpenOption.READ)) {
        java.nio.ByteBuffer buf = java.nio.ByteBuffer.allocate(64);
        while (file.read(buf) != -1) {
            buf.flip();
            while (buf.hasRemaining()) {
                socket.write(buf);
            }
            buf.clear();
        }
        socket.shutdownOutput();
    }
}
```

把两边收进一个可运行的类，服务端先启动，稍后再连。收到文件后回一条短消息，客户端把它打印出来。

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;
import java.util.concurrent.TimeUnit;

public class BlockingTransferLab {
    static final int PORT = 19090;

    public static void send() throws IOException {
        try (SocketChannel socket = SocketChannel.open(new InetSocketAddress("127.0.0.1", PORT));
             FileChannel file = FileChannel.open(Paths.get("/tmp/nio-lab/manifest.txt"), StandardOpenOption.READ)) {
            ByteBuffer buf = ByteBuffer.allocate(64);
            while (file.read(buf) != -1) {
                buf.flip();
                while (buf.hasRemaining()) {
                    socket.write(buf);
                }
                buf.clear();
            }
            socket.shutdownOutput();
            int n;
            while ((n = socket.read(buf)) != -1) {
                buf.flip();
                System.out.println(StandardCharsets.UTF_8.decode(buf).toString());
                buf.clear();
                if (n == 0) {
                    break;
                }
            }
        }
    }

    public static void serve() throws IOException {
        try (ServerSocketChannel listener = ServerSocketChannel.open();
             FileChannel file = FileChannel.open(
                     Paths.get("/tmp/nio-lab/inbox.bin"),
                     StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
            listener.bind(new InetSocketAddress(PORT));
            try (SocketChannel client = listener.accept()) {
                ByteBuffer buf = ByteBuffer.allocate(64);
                while (client.read(buf) != -1) {
                    buf.flip();
                    file.write(buf);
                    buf.clear();
                }
                buf.put("已收下清单".getBytes(StandardCharsets.UTF_8));
                buf.flip();
                client.write(buf);
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Thread hub = new Thread(() -> {
            try {
                serve();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }, "hub");
        hub.start();
        TimeUnit.MILLISECONDS.sleep(200);
        send();
        hub.join();
    }
}
```

这仍然是阻塞模型：没有客户端时 `accept` 不返回；客户端停住不关输出时，`read` 也不返回。连接数上来之后，不能靠无限加线程硬扛。

## 使用非阻塞式IO完成网络通信

非阻塞加上选择器之后，线程只在至少有一条通道就绪时被叫醒。客户端连上后把通道登记成“可读”，以后这条连接有字节到达才会进入读分支。

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.time.LocalTime;
import java.util.Iterator;
import java.util.concurrent.TimeUnit;

public class SelectorLab {
    static final int PORT = 19091;

    public static void send(String payload) throws IOException {
        try (SocketChannel socket = SocketChannel.open(new InetSocketAddress("127.0.0.1", PORT))) {
            socket.configureBlocking(false);
            ByteBuffer buf = ByteBuffer.wrap(payload.getBytes(StandardCharsets.UTF_8));
            while (buf.hasRemaining()) {
                socket.write(buf);
            }
        }
    }

    public static void serve() throws IOException {
        try (ServerSocketChannel listener = ServerSocketChannel.open();
             Selector selector = Selector.open()) {
            listener.configureBlocking(false);
            listener.bind(new InetSocketAddress(PORT));
            listener.register(selector, SelectionKey.OP_ACCEPT);
            while (selector.select(500) > 0) {
                Iterator<SelectionKey> ready = selector.selectedKeys().iterator();
                while (ready.hasNext()) {
                    SelectionKey key = ready.next();
                    ready.remove();
                    if (!key.isValid()) {
                        continue;
                    }
                    if (key.isAcceptable()) {
                        SocketChannel client = listener.accept();
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_READ);
                    } else if (key.isReadable()) {
                        SocketChannel client = (SocketChannel) key.channel();
                        ByteBuffer buf = ByteBuffer.allocate(128);
                        int n = client.read(buf);
                        if (n < 0) {
                            key.cancel();
                            client.close();
                            continue;
                        }
                        buf.flip();
                        System.out.println(Thread.currentThread().getName() + " "
                                + StandardCharsets.UTF_8.decode(buf));
                    }
                }
            }
        }
    }

    public static void main(String[] args) throws Exception {
        Thread hub = new Thread(() -> {
            try {
                serve();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }, "hub");
        hub.start();
        TimeUnit.MILLISECONDS.sleep(200);
        for (int i = 0; i < 3; i++) {
            send("班次" + i + "@" + LocalTime.now());
        }
        TimeUnit.MILLISECONDS.sleep(300);
        hub.interrupt();
    }
}
```

几个容易写错的点：

- 处理完的键必须从已选集合删掉，否则下一轮还会看到它
- 读到 -1 表示对方关闭了输出，要取消登记并关掉通道，不然选择器会一直认为它可读
- `write` 在非阻塞下可能一次写不完，正式服务端还应登记 `OP_WRITE`，有空间再继续写
- `select` 返回的是就绪集合，不是“所有已注册通道”

## 使用非阻塞式IO制作聊天室

把上面的读分支改成转发，就成了一个最小聊天室：谁发来一行，服务器就抄给其它仍然打开的通道。客户端用扫描器从控制台取一行。

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;

public class ChatHub {
    public static void main(String[] args) throws IOException {
        try (ServerSocketChannel listener = ServerSocketChannel.open();
             Selector selector = Selector.open()) {
            listener.bind(new InetSocketAddress(19092));
            listener.configureBlocking(false);
            listener.register(selector, SelectionKey.OP_ACCEPT);
            ByteBuffer buf = ByteBuffer.allocate(256);
            while (selector.select() > 0) {
                Iterator<SelectionKey> ready = selector.selectedKeys().iterator();
                while (ready.hasNext()) {
                    SelectionKey key = ready.next();
                    ready.remove();
                    if (key.isAcceptable()) {
                        SocketChannel newbie = listener.accept();
                        newbie.configureBlocking(false);
                        newbie.register(selector, SelectionKey.OP_READ);
                    } else if (key.isReadable()) {
                        SocketChannel speaker = (SocketChannel) key.channel();
                        buf.clear();
                        int n = speaker.read(buf);
                        if (n <= 0) {
                            key.cancel();
                            speaker.close();
                            continue;
                        }
                        buf.flip();
                        for (SelectionKey other : selector.keys()) {
                            if (other.channel() instanceof SocketChannel && other.channel() != speaker) {
                                ByteBuffer copy = ByteBuffer.allocate(buf.remaining());
                                copy.put(buf.duplicate());
                                copy.flip();
                                ((SocketChannel) other.channel()).write(copy);
                            }
                        }
                    }
                }
            }
        }
    }
}
```

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Scanner;

public class ChatMember {
    public static void main(String[] args) throws Exception {
        try (SocketChannel socket = SocketChannel.open(new InetSocketAddress("127.0.0.1", 19092));
             Scanner scanner = new Scanner(System.in)) {
            socket.configureBlocking(false);
            ByteBuffer buf = ByteBuffer.allocate(256);
            while (scanner.hasNextLine()) {
                String line = scanner.nextLine();
                buf.clear();
                buf.put(line.getBytes(StandardCharsets.UTF_8));
                buf.flip();
                while (buf.hasRemaining()) {
                    socket.write(buf);
                }
            }
        }
    }
}
```

要同时跑两个成员，需要两个进程或允许同一项目并行启动。服务器只负责转发，它自己不解析昵称。非阻塞写如果对方接收窗口满了，这里的 `write` 仍可能没写完，生产代码要把剩余字节挂到该键的附件上，等 `OP_WRITE`。练习时先保证“能转发给其他人”，再补背压。

这个客户端是写多读少的演示：键盘输入被送出，别人的回复需要另开一条读循环才能打印。面试里只要讲清登记、就绪、转发三步即可。

## 管道（Pipe）

管道是同一台虚拟机里两条线程之间的单向连接。一端叫 sink，负责写；另一端叫 source，负责读。数据写进 sink，才能从 source 取出，不能反向。

```java
import java.nio.ByteBuffer;
import java.nio.channels.Pipe;
import java.nio.charset.StandardCharsets;

public class PipeLab {
    public static void main(String[] args) throws Exception {
        Pipe pipe = Pipe.open();
        ByteBuffer outgoing = ByteBuffer.allocate(64);
        outgoing.put("潮位正常".getBytes(StandardCharsets.UTF_8));
        outgoing.flip();
        try (Pipe.SinkChannel sink = pipe.sink();
             Pipe.SourceChannel source = pipe.source()) {
            while (outgoing.hasRemaining()) {
                sink.write(outgoing);
            }
            ByteBuffer incoming = ByteBuffer.allocate(64);
            int n = source.read(incoming);
            incoming.flip();
            System.out.println(StandardCharsets.UTF_8.decode(incoming).toString() + " bytes=" + n);
        }
    }
}
```

写之前要翻转，让上限停在有效字节上。读则使用另一块缓冲，避免和写缓冲的指针缠在一起。管道不是网络协议，也不能跨进程；跨进程仍要用套接字或文件。它适合在选择器教程里演示“一个线程写、另一个线程读”，因为 source 可以注册成可读。
