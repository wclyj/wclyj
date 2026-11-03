# NIO深入

## BIO

先看阻塞式套接字。服务端要先等一个人连上来，连上之后再等这个人把字节写进来。两处等待都会把当前线程按住。

客户端只发一行就返回：

```java
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class BoardClient {
    public static void main(String[] args) throws Exception {
        try (Socket socket = new Socket("127.0.0.1", 7077)) {
            socket.getOutputStream().write("泊位已空出".getBytes(StandardCharsets.UTF_8));
        }
    }
}
```

服务端如果把监听套接字放进死循环里面反复创建，第二个客户端会因为端口已被占用而失败。监听对象应该放到循环外，只让 `accept` 重复执行。

```java
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.StandardCharsets;

public class BoardServer {
    public static void main(String[] args) throws Exception {
        byte[] buf = new byte[64];
        try (ServerSocket server = new ServerSocket()) {
            server.bind(new InetSocketAddress(7077));
            while (true) {
                System.out.println("等待下一位写入者");
                Socket socket = server.accept();
                System.out.println("已经接上");
                int n = socket.getInputStream().read(buf);
                System.out.println("收到:" + new String(buf, 0, Math.max(n, 0), StandardCharsets.UTF_8));
                socket.close();
            }
        }
    }
}
```

先启动服务端，再启动客户端，能看到先卡住在接入，接入之后再卡住在读取，最后打出“泊位已空出”。线程在这两段时间里都不能去接待别人。空闲连接一多，线程就堆在等待上，所以才会想能不能只用一条线程把所有连接轮询一遍。

## BIO怎么改成非阻塞

阻塞点就在这两行：没有人连接时 `accept` 不返回；连接了但不发字节时 `read` 不返回。

```java
while (true) {
    Socket socket = server.accept();
    socket.getInputStream().read(buf);
}
```

设想一个人只打开页面、不再发请求。如果线程被这名用户按住，后面的人就进不来。为每个空闲用户单开线程也不划算，线程本身要占栈，还要参与调度。于是会希望“现在没有人就立刻返回，我去看看别的连接”。

旧的 `ServerSocket` 没有真正的非阻塞接入。即便写成下面这种假想配置，也只是注释，运行时并不存在这个方法，调用仍会堵住。

```java
// 旧套接字做不到：server.setNonBlocking()
```

用一份伪代码把“想要的轮询”写出来，就能看出缺什么。列表里保存已经接上的套接字；这一轮没人接入，就回头扫列表，看谁已经写来了字节。

```java
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.ArrayList;
import java.util.List;

public class PollingSketch {
    public static void main(String[] args) throws Exception {
        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress(7077));
        List<Socket> live = new ArrayList<>();
        byte[] buf = new byte[64];
        while (true) {
            Socket arrived = null; // 真实的 accept 在这里会一直等
            if (arrived == null) {
                System.out.println("这一轮没有新连接");
                for (Socket item : live) {
                    int n = 0; // 真实的 read 在没数据时也会一直等
                    if (n > 0) {
                        System.out.println(new String(buf, 0, n));
                    }
                }
            } else {
                live.add(arrived);
            }
        }
    }
}
```

问题不在循环怎么写，而在这两次调用根本不肯马上返回。所以阻塞模型里做不到“问一句就走”。

## NIO

新接口提供了可以立刻返回的通道。创建和绑定看起来和原来差不多，只是类型从套接字换成了通道。

```java
ServerSocketChannel hub = ServerSocketChannel.open();
hub.bind(new InetSocketAddress(7077));
```

新建出来的通道默认仍会阻塞。要轮询，必须先关掉阻塞模式。

```java
hub.configureBlocking(false);
```

接入也换成通道。非阻塞之下，此刻没有新连接时 `accept` 返回空引用，而不是把线程按住。已经接上的 `SocketChannel` 同样要关掉阻塞，否则对它的 `read` 仍会等待。

```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class BoardHub {
    public static void main(String[] args) throws Exception {
        ServerSocketChannel hub = ServerSocketChannel.open();
        hub.bind(new InetSocketAddress(7077));
        hub.configureBlocking(false);
        List<SocketChannel> live = new ArrayList<>();
        ByteBuffer buf = ByteBuffer.allocate(64);
        while (true) {
            SocketChannel arrived = hub.accept();
            if (arrived == null) {
                TimeUnit.MILLISECONDS.sleep(200);
                for (SocketChannel item : live) {
                    buf.clear();
                    int n = item.read(buf);
                    if (n > 0) {
                        buf.flip();
                        System.out.println(StandardCharsets.UTF_8.decode(buf).toString());
                    }
                }
            } else {
                arrived.configureBlocking(false);
                live.add(arrived);
            }
        }
    }
}
```

没有人时，线程睡一小会儿再扫，不会卡死在接入上。有字节时 `read` 返回长度；暂时没字节时返回 0。对方关掉连接会返回 -1，这时应把通道从列表删掉并关闭，否则下一轮还会去读一个已经失效的描述符。

这种手写轮询能说明非阻塞的含义，但连接一多，大部分循环都在访问还没数据的通道，处理器时间花在空转上。下一步才是选择器：把通道登记进去，没有就绪事件时线程真正休眠，而不是自己每隔 200 毫秒醒一次。
