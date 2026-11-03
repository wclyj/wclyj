# Volatile 不保证原子性

## 前言

JMM 规定线程不能直接改主内存。每个线程先把共享变量拷进自己的工作内存，改完再写回去。

于是可能出现这种交错：线程甲已经在本地把 `quota` 加一，但还没写回；线程乙读到的仍是旧值，也在本地加一。两个人都写回时，后写的覆盖先写的，一次加一丢失。这是可见性延迟，也是复合动作不原子。

## 原子性

原子指中间不能被别的线程插进来。一件事情要么整个做完，要么整个没做，不能只留下半截。数据库事务里的原子性是同一个词：转账不能只扣了付款方、收款方却没加上。

`volatile` 保证的是单次读、单次写的可见性和有序性。`quota++` 是读、改、写三步，不在它的保证范围内。

## 代码测试

用 15 条线程，每条把计数加 800 次。若自增是原子的，结果应是 `15 * 800 = 12000`。

```java
QuotaBox box = new QuotaBox();
for (int i = 0; i < 15; i++) {
    new Thread(() -> {
        for (int j = 0; j < 800; j++) {
            box.bump();
        }
    }, "adder-" + i).start();
}
```

主线程要等这 15 条线程结束再读结果。`Thread.activeCount()` 在没有其他框架线程时，通常至少还有 main 自己。判断写成大于 2，是把 main 和可能存在的后台线程算进去；更稳的做法是把每条线程收进列表再 `join`。下面两种都给出，示例用 `join`，避免机器上线程数不是 2 时循环停不下来。

```java
// 教学演示里常见的让步自旋。线程数不稳定时不要用它当唯一结束条件。
while (Thread.activeCount() > 2) {
    Thread.yield();
}
```

完整代码：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * bump 只有 volatile，没有原子自增。
 * bumpAtomic 用 AtomicInteger，结果应稳定等于 12000。
 */
class QuotaBox {
    volatile int quota = 0;
    AtomicInteger atomicQuota = new AtomicInteger();

    public void bump() {
        quota++;
    }

    public void bumpAtomic() {
        atomicQuota.getAndIncrement();
    }
}

public class VolatileBumpDemo {
    public static void main(String[] args) throws InterruptedException {
        QuotaBox box = new QuotaBox();
        List<Thread> workers = new ArrayList<>();
        for (int i = 0; i < 15; i++) {
            Thread worker = new Thread(() -> {
                for (int j = 0; j < 800; j++) {
                    box.bump();
                    box.bumpAtomic();
                }
            }, "adder-" + i);
            workers.add(worker);
            worker.start();
        }
        for (Thread worker : workers) {
            worker.join();
        }
        System.out.println("volatile quota = " + box.quota);
        System.out.println("atomic quota   = " + box.atomicQuota.get());
    }
}
```

多次运行，`atomic quota` 总是 12000，`volatile quota` 往往是 11000 多、10000 多，而且每次不同。可见性已经打开，原子性仍然没有。

## 为什么出现数值丢失

每个线程写回主内存时，可能覆盖别人已经加上的那一次。把 `quota++` 反编译，能看见它根本不是一条指令。

```java
public class BumpByte {
    volatile int quota = 0;
    public void bump() {
        quota++;
    }
}
```

`javap -c -p BumpByte` 里，`bump` 大致是：

```text
public void bump();
  Code:
     0: aload_0
     1: dup
     2: getfield      #2   // Field quota:I
     5: iconst_1
     6: iadd
     7: putfield      #2   // Field quota:I
    10: return
```

IDEA 里可以用 External Tools 配一条 `javap -c -p $FileClass$`，在 class 上右键查看。Maven 工程要先编译，否则找不到 class。Spring Boot 多模块时，class 在 `target/classes` 下，命令的工作目录要对准模块。

`quota++` 被拆成三步：

- `getfield`：按对象引用读出字段的当前 int
- `iconst_1` 然后 `iadd`：在操作数栈上加一
- `putfield`：把和写回字段

假设没有锁。三条线程可以都执行完 `getfield`，都拿到 10。然后各自 `iadd` 得到 11，再排队 `putfield`。三次写回去的都是 11，不是 13。`volatile` 会让每次 `putfield` 对别人可见，但可见的是一个已经算错的 11。别人来不及重新 `getfield`，因为它们的加一在写回之前就已经算完了。

所以丢数不是“volatile 没把新值通知出去”，而是“通知出去的时候，别的线程手里的加法已经用旧值算完了”。

## 如何解决

给方法加上 `synchronized`，同一时刻只允许一个线程执行整段读改写：

```java
public synchronized void bump() {
    quota++;
}
```

15 条线程、每条 800 次，结果稳定是 12000。锁把三步收成临界区，可见性也由锁的释放和获取一起保证。为了一个自增使用监视器，并发高时线程会在锁上排队，成本比一条 CAS 大。

## 其它解决方法

自增这种单变量读改写，更合适的是原子类。`AtomicInteger.getAndIncrement` 在循环里做 CAS，失败就重读再试，不需要把整个方法互斥。

```java
AtomicInteger atomicQuota = new AtomicInteger();

public void bumpAtomic() {
    atomicQuota.getAndIncrement();
}
```

上面的完整示例里，同一个循环既调用 `bump` 又调用 `bumpAtomic`。前者飘在 12000 以下，后者每次都是 12000。两者的可见性都没问题，差别只在读改写是不是一步完成。

若既要原子，又要和别的字段一起成功或一起失败，单个 `AtomicInteger` 就不够了，那时再回到锁，或者把多个字段收进一个不可变对象做一次 `AtomicReference` 替换。

## 字节码指令表

下面按操作码顺序整理 `bump` 附近会遇到的指令。助记符是规范里的名字，说明换成便于口算的说法。查表时先看助记符，再看它动的是操作数栈还是局部变量表。

| 操作码 | 助记符 | 在做什么 |
| --- | --- | --- |
| 0x00 | nop | 什么都不做，占一条指令的位置 |
| 0x01 | aconst_null | 把空引用压进操作数栈 |
| 0x02 | iconst_m1 | 压入整数 -1 |
| 0x03 | iconst_0 | 压入整数 0 |
| 0x04 | iconst_1 | 压入整数 1，自增加的就是它 |
| 0x05 | iconst_2 | 压入整数 2 |
| 0x06 | iconst_3 | 压入整数 3 |
| 0x07 | iconst_4 | 压入整数 4 |
| 0x08 | iconst_5 | 压入整数 5 |
| 0x09 | lconst_0 | 压入 long 0 |
| 0x0a | lconst_1 | 压入 long 1 |
| 0x0b | fconst_0 | 压入 float 0 |
| 0x0c | fconst_1 | 压入 float 1 |
| 0x0d | fconst_2 | 压入 float 2 |
| 0x0e | dconst_0 | 压入 double 0 |
| 0x0f | dconst_1 | 压入 double 1 |
| 0x10 | bipush | 把后面一个字节当成 int 压栈，范围 -128 到 127 |
| 0x11 | sipush | 把后面两个字节当成 short 再扩成 int 压栈 |
| 0x12 | ldc | 从常量池取 int、float 或字符串压栈 |
| 0x13 | ldc_w | 同上，常量池下标用两个字节 |
| 0x14 | ldc2_w | 从常量池取 long 或 double，占两个栈槽 |
| 0x15 | iload | 按后面的下标，把局部变量里的 int 压栈 |
| 0x16 | lload | 把局部变量里的 long 压栈 |
| 0x17 | fload | 把局部变量里的 float 压栈 |
| 0x18 | dload | 把局部变量里的 double 压栈 |
| 0x19 | aload | 把局部变量里的引用压栈。实例方法开头的 aload_0 就是 this |
| 0x1a | iload_0 | 读取 0 号局部变量的 int |
| 0x1b | iload_1 | 读取 1 号局部变量的 int |
| 0x1c | iload_2 | 读取 2 号局部变量的 int |
| 0x1d | iload_3 | 读取 3 号局部变量的 int |
| 0x1e | lload_0 | 读取 0 号槽的 long |
| 0x1f | lload_1 | 读取 1 号槽的 long |
| 0x20 | lload_2 | 读取 2 号槽的 long |
| 0x21 | lload_3 | 读取 3 号槽的 long |
| 0x22 | fload_0 | 读取 0 号槽的 float |
| 0x23 | fload_1 | 读取 1 号槽的 float |
| 0x24 | fload_2 | 读取 2 号槽的 float |
| 0x25 | fload_3 | 读取 3 号槽的 float |
| 0x26 | dload_0 | 读取 0 号槽的 double |
| 0x27 | dload_1 | 读取 1 号槽的 double |
| 0x28 | dload_2 | 读取 2 号槽的 double |
| 0x29 | dload_3 | 读取 3 号槽的 double |
| 0x2a | aload_0 | 读取 0 号槽的引用，实例方法里就是 this |
| 0x2b | aload_1 | 读取 1 号槽的引用 |
| 0x2c | aload_2 | 读取 2 号槽的引用 |
| 0x2d | aload_3 | 读取 3 号槽的引用 |
| 0x2e | iaload | 弹出下标和 int 数组，把那个元素压回去 |
| 0x2f | laload | 读取 long 数组元素 |
| 0x30 | faload | 读取 float 数组元素 |
| 0x31 | daload | 读取 double 数组元素 |
| 0x32 | aaload | 读取引用数组元素 |
| 0x33 | baload | 读取 boolean 或 byte 数组元素 |
| 0x34 | caload | 读取 char 数组元素 |
| 0x35 | saload | 读取 short 数组元素 |
| 0x36 | istore | 把栈顶 int 写进指定局部变量 |
| 0x37 | lstore | 把栈顶 long 写进局部变量 |
| 0x38 | fstore | 把栈顶 float 写进局部变量 |
| 0x39 | dstore | 把栈顶 double 写进局部变量 |
| 0x3a | astore | 把栈顶引用写进局部变量 |
| 0x3b | istore_0 | 栈顶 int 写入 0 号槽 |
| 0x3c | istore_1 | 栈顶 int 写入 1 号槽 |
| 0x3d | istore_2 | 栈顶 int 写入 2 号槽 |
| 0x3e | istore_3 | 栈顶 int 写入 3 号槽 |
| 0x3f | lstore_0 | 栈顶 long 写入 0 号槽 |
| 0x40 | lstore_1 | 栈顶 long 写入 1 号槽 |
| 0x41 | lstore_2 | 栈顶 long 写入 2 号槽 |
| 0x42 | lstore_3 | 栈顶 long 写入 3 号槽 |
| 0x43 | fstore_0 | 栈顶 float 写入 0 号槽 |
| 0x44 | fstore_1 | 栈顶 float 写入 1 号槽 |
| 0x45 | fstore_2 | 栈顶 float 写入 2 号槽 |
| 0x46 | fstore_3 | 栈顶 float 写入 3 号槽 |
| 0x47 | dstore_0 | 栈顶 double 写入 0 号槽 |
| 0x48 | dstore_1 | 栈顶 double 写入 1 号槽 |
| 0x49 | dstore_2 | 栈顶 double 写入 2 号槽 |
| 0x4a | dstore_3 | 栈顶 double 写入 3 号槽 |
| 0x4b | astore_0 | 栈顶引用写入 0 号槽 |
| 0x4c | astore_1 | 栈顶引用写入 1 号槽 |
| 0x4d | astore_2 | 栈顶引用写入 2 号槽 |
| 0x4e | astore_3 | 栈顶引用写入 3 号槽 |
| 0x4f | iastore | 把 int 写进数组的指定下标 |
| 0x50 | lastore | 把 long 写进数组 |
| 0x51 | fastore | 把 float 写进数组 |
| 0x52 | dastore | 把 double 写进数组 |
| 0x53 | aastore | 把引用写进数组 |
| 0x54 | bastore | 把 boolean 或 byte 写进数组 |
| 0x55 | castore | 把 char 写进数组 |
| 0x56 | sastore | 把 short 写进数组 |
| 0x57 | pop | 丢掉栈顶一个槽。long 和 double 不能用它 |
| 0x58 | pop2 | 丢掉两个槽，或丢掉一个 long/double |
| 0x59 | dup | 复制栈顶一槽，再压回去 |
| 0x5a | dup_x1 | 复制栈顶，并把它插到下一槽的下面 |
| 0x5b | dup_x2 | 复制栈顶，并插到下面两槽之后 |
| 0x5c | dup2 | 复制栈顶两个槽，或复制一个 long/double |
| 0x5d | dup2_x1 | dup_x1 的双字版本 |
| 0x5e | dup2_x2 | dup_x2 的双字版本 |
| 0x5f | swap | 交换栈顶两槽，不能用于 long 或 double |
| 0x60 | iadd | 两个 int 相加，和留在栈上。quota++ 用它 |
| 0x61 | ladd | 两个 long 相加 |
| 0x62 | fadd | 两个 float 相加 |
| 0x63 | dadd | 两个 double 相加 |
| 0x64 | isub | 两个 int 相减 |
| 0x65 | lsub | 两个 long 相减 |
| 0x66 | fsub | 两个 float 相减 |
| 0x67 | dsub | 两个 double 相减 |
| 0x68 | imul | 两个 int 相乘 |
| 0x69 | lmul | 两个 long 相乘 |
| 0x6a | fmul | 两个 float 相乘 |
| 0x6b | dmul | 两个 double 相乘 |
| 0x6c | idiv | 两个 int 相除 |
| 0x6d | ldiv | 两个 long 相除 |
| 0x6e | fdiv | 两个 float 相除 |
| 0x6f | ddiv | 两个 double 相除 |
| 0x70 | irem | int 取余 |
| 0x71 | lrem | long 取余 |
| 0x72 | frem | float 取余 |
| 0x73 | drem | double 取余 |
| 0x74 | ineg | int 取相反数 |
| 0x75 | lneg | long 取相反数 |
| 0x76 | fneg | float 取相反数 |
| 0x77 | dneg | double 取相反数 |
| 0x78 | ishl | int 左移 |
| 0x79 | lshl | long 左移 |
| 0x7a | ishr | int 算术右移，保留符号 |
| 0x7b | lshr | long 算术右移 |
| 0x7c | iushr | int 逻辑右移，高位补 0 |
| 0x7d | lushr | long 逻辑右移 |
| 0x7e | iand | 两个 int 按位与 |
| 0x7f | land | 两个 long 按位与 |
| 0x80 | ior | 两个 int 按位或 |
| 0x81 | lor | 两个 long 按位或 |
| 0x82 | ixor | 两个 int 按位异或 |
| 0x83 | lxor | 两个 long 按位异或 |
| 0x84 | iinc | 局部变量直接加一个常量，不再经过栈。编译器有时用它实现 i++ |
| 0x85 | i2l | int 扩成 long |
| 0x86 | i2f | int 转 float |
| 0x87 | i2d | int 转 double |
| 0x88 | l2i | long 截成 int |
| 0x89 | l2f | long 转 float |
| 0x8a | l2d | long 转 double |
| 0x8b | f2i | float 转 int |
| 0x8c | f2l | float 转 long |
| 0x8d | f2d | float 转 double |
| 0x8e | d2i | double 转 int |
| 0x8f | d2l | double 转 long |
| 0x90 | d2f | double 转 float |
| 0x91 | i2b | int 截成 byte |
| 0x92 | i2c | int 截成 char |
| 0x93 | i2s | int 截成 short |
| 0x94 | lcmp | 比较两个 long，把 -1、0 或 1 压栈 |
| 0x95 | fcmpl | 比较 float，遇到 NaN 时压 -1 |
| 0x96 | fcmpg | 比较 float，遇到 NaN 时压 1 |
| 0x97 | dcmpl | 比较 double，遇到 NaN 时压 -1 |
| 0x98 | dcmpg | 比较 double，遇到 NaN 时压 1 |
| 0x99 | ifeq | 栈顶 int 是 0 就跳转 |
| 0x9a | ifne | 栈顶 int 不是 0 就跳转 |
| 0x9b | iflt | 栈顶 int 小于 0 就跳转 |
| 0x9c | ifge | 栈顶 int 大于等于 0 就跳转 |
| 0x9d | ifgt | 栈顶 int 大于 0 就跳转 |
| 0x9e | ifle | 栈顶 int 小于等于 0 就跳转 |
| 0x9f | if_icmpeq | 两个 int 相等就跳转 |
| 0xa0 | if_icmpne | 两个 int 不等就跳转 |
| 0xa1 | if_icmplt | 上面的 int 更小就跳转 |
| 0xa2 | if_icmpge | 上面的 int 大于等于下面的就跳转 |
| 0xa3 | if_icmpgt | 上面的 int 更大就跳转 |
| 0xa4 | if_icmple | 上面的 int 小于等于下面的就跳转 |
| 0xa5 | if_acmpeq | 两个引用指向同一对象就跳转 |
| 0xa6 | if_acmpne | 两个引用不是同一对象就跳转 |
| 0xa7 | goto | 无条件跳到给出的偏移 |
| 0xa8 | jsr | 跳到子程序，并把返回地址压栈。新代码很少用 |
| 0xa9 | ret | 从局部变量取出返回地址再跳回去 |
| 0xaa | tableswitch | switch 的跳转表，case 值连续 |
| 0xab | lookupswitch | switch 的稀疏跳转，case 值不连续 |
| 0xac | ireturn | 方法返回一个 int |
| 0xad | lreturn | 方法返回一个 long |
| 0xae | freturn | 方法返回一个 float |
| 0xaf | dreturn | 方法返回一个 double |
| 0xb0 | areturn | 方法返回一个引用 |
| 0xb1 | return | 方法不返回值 |
| 0xb2 | getstatic | 读取静态字段并压栈 |
| 0xb3 | putstatic | 给静态字段赋值 |
| 0xb4 | getfield | 读取实例字段。volatile 读也从这里进入内存屏障 |
| 0xb5 | putfield | 写实例字段。volatile 写会把之前的普通写一并发布出去 |
| 0xb6 | invokevirtual | 按对象的实际类型调用实例方法 |
| 0xb7 | invokespecial | 调用构造方法、私有方法或超类方法 |
| 0xb8 | invokestatic | 调用静态方法 |
| 0xb9 | invokeinterface | 调用接口方法 |
| 0xba | invokedynamic | 调用引导方法给出的动态目标，lambda 会用到 |
| 0xbb | new | 分配对象，把引用压栈，此时构造还没跑 |
| 0xbc | newarray | 分配基本类型数组 |
| 0xbd | anewarray | 分配引用类型数组 |
| 0xbe | arraylength | 取数组长度压栈 |
| 0xbf | athrow | 把栈顶异常抛出去 |
| 0xc0 | checkcast | 检查引用能不能当成目标类型，失败抛 ClassCastException |
| 0xc1 | instanceof | 类型匹配压 1，否则压 0 |
| 0xc2 | monitorenter | 进入对象监视器，对应同步块开始 |
| 0xc3 | monitorexit | 退出对象监视器，对应同步块结束 |
| 0xc4 | wide | 加宽后面那条指令的局部变量下标 |
| 0xc5 | multianewarray | 分配多维数组，各维长度已经在栈上 |
| 0xc6 | ifnull | 栈顶引用是空就跳转 |
| 0xc7 | ifnonnull | 栈顶引用非空就跳转 |
| 0xc8 | goto_w | 使用四字节偏移的无条件跳转 |
| 0xc9 | jsr_w | 使用四字节偏移的 jsr |

读 `bump` 时真正要盯住的是 `getfield`、`iadd`、`putfield` 这三步之间可以被别的线程插进来。表里其余指令用来对照：一旦反编译结果里出现 `monitorenter`，说明已经用同步块把这三步包住了。

## 一次丢失是怎么发生的

把 `quota` 的初值设成 40，只留下甲、乙两个线程各做一次 `bump`。若两次加一都完整发生，结果应是 42。下面按时间写出一种会得到 41 的交错。局部变量名只用口头称呼，对应的就是操作数栈上的那份临时值。

1. 甲执行 `aload_0` 和 `getfield`，从主内存读到 40，栈顶现在是 40。
2. 调度器把甲换下。乙同样 `getfield`，也读到 40。
3. 乙执行 `iconst_1`、`iadd`，栈上变成 41，再 `putfield` 写回。因为字段是 volatile 的，这次写会让甲缓存里的旧行失效。
4. 甲恢复。它的操作数栈上仍留着刚才的 40，`iadd` 只把这个栈顶加一，得到 41，再 `putfield`。volatile 保证这次 41 能被别人看见，但 41 覆盖了乙已经写回的 41，乙那一次加法从结果里消失。

所以最终是 41，不是 42。问题出在第 4 步没有重新 `getfield`。`volatile` 不会在你已经把旧值放进操作数栈之后，自动帮你再读一次。

把同一段改成 `synchronized` 之后，字节码在 `getfield` 之前出现 `monitorenter`，在 `putfield` 之后出现 `monitorexit`。乙要进方法必须等甲把监视器放开。甲写回 41 并退出后，乙再读，读到的是 41，加上一变成 42。三步不再被拆开。

`AtomicInteger.getAndIncrement` 不生成这三步 Java 字节码去碰字段，而是调用 Unsafe 的循环：每次重新取内存里的当前值，CAS 成功才离开。乙已经写成 41 时，甲手里的预期 40 对不上，循环回到开头，重新读到 41，再写成 42。可见性由 volatile 的 `value` 提供，原子性由 CAS 提供，两件事缺一不可。

## 什么时候 volatile 仍然够用

下面这些写法是单次写，不是读改写，`volatile` 就能同时给可见性和有序性：

- 状态标志：一个线程写 `volatile boolean stopped = true`，工作线程在循环里读它。写是一次赋值，读是一次取值，没有“读出来再加”的窗口。
- 一次性发布：配置对象构造完成后，用一次 volatile 写把引用交给其他线程。读者只要看见非空引用，就能看见构造里的字段，这条靠的是禁止重排，不是靠原子自增。
- 双重检查单例里的那个静态引用。需要原子的是“只有一个线程执行构造”，那部分用 `synchronized`；`volatile` 负责不把未构造完的引用发出去。

下面这些不要指望 `volatile` 单独完成：

- `count++`、`count += n`、`count = count * 2`，都是读改写。
- 先检查再行动，例如 `if (tickets > 0) tickets--`。检查和减少之间别人可以插进来。
- 两个字段要一起变，例如扣库存的同时还要加一条流水。两个 volatile 写之间仍然有窗口，需要锁，或者收成一个对象做一次原子替换。

## 和字节码表对一下号

实例方法的开头几乎总是 `aload_0`，把 `this` 压栈。`dup` 再复制一份，是因为 `getfield` 会消费栈顶的对象引用，而后面的 `putfield` 还需要一份同样的引用。所以自增不是“一个局部变量加一”那么简单：字段不在局部变量表里，必须先装到栈上。

编译器对局部变量的 `i++` 有时会收成一条 `iinc`，那条指令不经过操作数栈，在单线程里是原子的。字段不行。字段可能被别的线程同时看，JVM 必须发出 `getfield` 和 `putfield`，中间自然露出空档。看到反编译里字段自增变成了 `getfield / iadd / putfield`，就可以直接回答：volatile 盖不住这个空档。

若反编译里出现 `invokestatic` 指向 `AtomicInteger.getAndIncrement`，空档被挪进了 JDK 的 CAS 循环。若出现成对的 `monitorenter` 和 `monitorexit`，空档被锁包住。两种都对，选择取决于临界区里还有没有别的必须一起完成的步骤。只有一个计数器时优先原子类；计数器和另一个结构必须同时成功时用锁。

再补一条容易混的边界：`volatile` 的单次写本身是原子的。给一个 `volatile int` 赋值 80，不会出现别的线程读到半个 int。丢数只发生在“先读再算再写”这种由多条字节码组成的动作上。long 和 double 的非 volatile 普通写，在很老的规范里允许撕裂成两次 32 位；标成 volatile 之后，一次读写也是原子的。所以面试里要分开说：volatile 让单次读写原子且可见，不让复合运算原子。把这两句分开背，就不容易在追问里答串到一起。这一点务必记牢。
