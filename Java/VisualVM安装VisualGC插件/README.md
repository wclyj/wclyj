# 给 VisualVM 装上分代视图

## 它能看什么

JDK 自带的 Java VisualVM 可以挂到本机或远程进程上，看线程、堆直方图、类加载数量。装上 VisualGC 之后，还能把新生代、老年代画成随时间变化的条带，用来确认一次分配风暴有没有把 Eden 打满、对象有没有熬过第一次回收。

## 从哪里打开

在 JDK 安装目录的 `bin` 下可以找到 `jvisualvm`（部分新版本随独立包发布，不再塞进 JDK）。命令行直接执行：

```bash
jvisualvm
```

左侧出现本地进程列表，就说明图形界面已经起来。双击目标进程即可挂上。

## 安装分代插件

VisualGC 用图形把堆的 Eden、两个 Survivor、老年代以及元空间的占用画出来，比只看一个总堆数字更容易判断“对象是不是刚从年轻代晋升”。

### 拿到与 JDK 匹配的插件包

打开 VisualVM 的插件中心页面，先按本机 JDK 的主版本选仓库。JDK 8 和 JDK 11 的插件二进制并不通用，下错版本会在安装时提示不兼容。找到 VisualGC 对应的 nbm 包，下载到本地。

插件文件可以先放到用户目录下 VisualVM 的配置区，例如：

```text
~/.visualvm/<版本>/
```

Windows 上常见位置在用户主目录的 `AppData/Roaming/VisualVM` 下。目录只是暂存，真正生效要在界面里再装一次。

### 在界面里装入

回到已经打开的 VisualVM：菜单选“工具 → 插件”，切到“已下载”，点“添加插件”，选中刚才的 nbm，按提示完成安装。装完后重新选中目标进程，页签里应出现 Visual GC。

## 用一段会撑满年轻代的程序验证

写一个只往列表里塞字节数组的小程序，让分配速度稳定、可预期：

```java
public class HeapPressure {
    public static void main(String[] args) throws Exception {
        List<byte[]> held = new ArrayList<>();
        while (true) {
            Thread.sleep(2);
            held.add(new byte[256 * 1024]);
        }
    }
}
```

启动参数把堆压小，方便在几秒内看到回收：

```bash
-Xms64m -Xmx64m
```

进程跑起来后，在 Visual GC 里看各区。开始时 Eden 快速上涨；过一小会儿 Survivor 一侧出现存活对象，说明年轻代已经做过至少一轮回收，一部分数组从 Eden 被复制了过去。如果列表一直强引用这些数组，后续还能看到老年代逐步抬升，直到抛出堆内存耗尽。

看图时不要只盯总堆：Eden 掉下去而 Survivor 升上来，才说明回收真的发生了，而不是进程还没开始分配。
