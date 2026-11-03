# 集群高并发下怎样生成全局唯一编号

## 问题

### 什么情况下必须要全局唯一的编号

服务被拆开之后，订单、消息、券、骑手任务都要有一个大家都能认的编号。数据拆到多库多表时，单库自增只在自己的库里唯一，拼到全局就会撞车。

金融流水、配送单、票务座位都靠这个编号做主键或幂等键。编号服务一旦重复或长时间不可用，后面的写入都会停。

### ID生成规则部分硬性要求

- 全局不重复
- 大致随时间变大。聚簇索引把新行写在叶子右侧时最省事，主键乱序会把页从中间拆开
- 必要时还要严格单调。版本号、收件箱位点这一类，后发出的号必须大于先发出的号。多数业务只要趋势递增就够了
- 不能让外人一眼看出今天的单量。纯连续的号既好遍历，也会泄露经营数据
- 号里最好能看出生成时刻，排查时不用再查一张时间表

这几条经常互相顶着。连续就容易被猜，不可猜就不可能严格单调。订单号可以在趋势递增的号外面再做一次混淆，对内仍有序，对外不规则。

### ID号生成系统的可用性要求

- 要能一直发号。调用方看到失败的比例要极低，单个节点挂了不能让所有业务停写
- 要快。发号如果要几十毫秒，会变成下单接口里最贵的一步
- 要扛突发。一瞬间十万个请求进来，也不能串行到把超时打满

可用性比“号有多漂亮”优先。发不出来号，业务主键就没了。

## 一般通用解决方案

### UUID

`UUID.randomUUID()` 在本地生成 128 位，写成十六进制是 8-4-4-4-12，共 36 个字符。不访问网络，重复概率低到业务里可以当成不会撞。

问题出在把它当主键：

- 无序。新行随机落在索引的各个叶子上，页被频繁拆分，插得不满的页变多
- 太长。二级索引的叶子还要再存一份主键，长度会放大到每条索引
- 只有唯一，没有趋势递增，更没有单调

官方也建议主键短而有序。随机主键在机械盘时代问题更明显，固态盘上随机写便宜了一些，页分裂和缓冲池浪费仍然在。能接受无序、又主要在应用里当关联键时，通用唯一标识仍然好用。

### 数据库自增主键

#### 单机

单库可以靠自增列发号。一种常见写法是用替换语句碰一个只有唯一键的单行表，再读刚刚生成的号。

`REPLACE` 按主键或唯一键判断：没有冲突就插入，有冲突就删掉旧行再插入，自增值照样往前走。

```sql
REPLACE INTO id_stub(stub) VALUES ('ticket');
SELECT LAST_INSERT_ID();
```

每次替换都让自增加一，于是号是唯一的、递增的，也是单调的。并发不高、只有一个写库时，这能用。

#### 集群分布式集群

把它扩成多台写库就别扭。号要靠步长错开，例如两台机器一个从 1 开始步长 2，一个从 2 开始步长 2。机器数一旦要变，步长和起点都要重算，线上已经发出的号还不能重。一百台写库再扩一台，几乎是在改一套已经散出去的规则。

每个号都要写一次数据库，延迟和吞吐都卡在这台发号库上。这和“低延迟、高吞吐”直接相反。发号库自己也要做高可用，复杂度并不比专门的发号服务低。

单库自增适合还没拆库的系统。确定要水平拆分时，应尽早把发号从业务库里拿出去。

### 基于Redis生成全局ID策略

#### 单机版

缓存执行命令的那条主路径是单线程的，`INCR` 和 `INCRBY` 不会被别的命令从中间插断。用一个键做计数器，就能得到不重复的递增号。`INCRBY` 用来一次跳过一段，方便按步长分给不同节点。

#### 集群分布式

多台缓存节点要像多台数据库一样错开起点和步长，并且给键设好持久化，不能把发号键做成会过期的缓存。假设四台节点，起点分别是 1、2、3、4，步长都是 4：

```text
A: 1, 5, 9, 13
B: 2, 6, 10, 14
C: 3, 7, 11, 15
D: 4, 8, 12, 16
```

号在全局上仍唯一，每一台本地也递增。代价是要维持这组起点，还要处理主从切换。从节点如果比主节点少一段增量，切换之后可能把旧号再发一遍，所以发号键要配合可靠的持久化和复制确认。

另一个现实问题是：只为了发号就部署并值守一整套缓存，运维成本和业务收益不成比例。缓存更适合已经因为别的原因部署了、吞吐又确实压满数据库的时候再拿来发号。

## 雪花算法

### 是什么

这是一套用 64 位整数拼出来的趋势递增号，最早用在从关系数据库迁到没有自增能力的分布式存储之后。一台机器每毫秒可以发一批，整体每秒几十万没有压力。

- 号里含时间，大体上后生成的更大
- 结果放得进一个长整型，十进制最多 19 位
- 不同机房、不同机器用不同的编号，避免撞号

它要满足的仍然是前面那些要求里最常见的三条：全局唯一、趋势递增、尽量短。若业务还要求外人猜不出单量，不要把这种号原样暴露在链接里。

### 结构

64 位分成四段。符号位占 1 位，时间占 41 位，机器占 10 位，序列占 12 位。

#### 第一部分

最高位是符号。1 表示负数。编号要用正数，所以这一位固定写 0。有的序列化把最高位拿去当别的标记，接号的时候不要把长整型当成无符号数随意右移。

#### 第二部分

接下来 41 位是毫秒时间，存的是“现在减去自定义纪元”，不是完整的纪元毫秒，否则位数不够。41 位能表示的毫秒数大约是 69.7 年。纪元如果设在系统上线附近，可用窗口就从那天开始算，而不是从 1970 年白占掉几十年。

#### 第三部分

10 位机器号最多区分 1024 台发号进程。常见切法是 5 位机房加 5 位机器，各能表示 0 到 31。两台进程如果配成同一个号，它们在同一毫秒生成的序列会撞车。机器号必须在部署时分配，不能每台都写 0。

#### 第四部分

12 位序列区分同一台机器同一毫秒里的多个号，范围是 0 到 4095，也就是一毫秒最多 4096 个。用尽之后不能在这一毫秒里继续从 0 再来，必须等到下一毫秒。

#### SnowFlake可以保证

时间在高位，所以整体随时间变大。机器号不同，同一毫秒的号也不会撞。它不保证多台机器之间严格单调：时钟快的那台会暂时发出更大的号。

### 实现

下面是一份独立实现，位布局和常见方案一致，纪元、字段名和等待方式是按这份笔记写的。同一毫秒内序列加一，加满就空转到下一毫秒。时钟相对上次发号倒退时直接拒绝，避免把旧时间配上旧序列，发出重复号。

```java
public class FlakeIds {
    private static final long EPOCH = 1_704_067_200_000L;
    private static final int SEQ_BITS = 12;
    private static final int WORKER_BITS = 5;
    private static final int ROOM_BITS = 5;

    private static final long SEQ_MASK = (1L << SEQ_BITS) - 1;
    private static final long WORKER_MASK = (1L << WORKER_BITS) - 1;
    private static final long ROOM_MASK = (1L << ROOM_BITS) - 1;

    private static final int WORKER_SHIFT = SEQ_BITS;
    private static final int ROOM_SHIFT = SEQ_BITS + WORKER_BITS;
    private static final int TIME_SHIFT = ROOM_SHIFT + ROOM_BITS;

    private final long roomId;
    private final long workerId;
    private long sequence;
    private long lastMillis = -1L;

    public FlakeIds(long roomId, long workerId) {
        if (roomId < 0 || roomId > ROOM_MASK || workerId < 0 || workerId > WORKER_MASK) {
            throw new IllegalArgumentException("room or worker out of range");
        }
        this.roomId = roomId;
        this.workerId = workerId;
    }

    public synchronized long next() {
        long now = System.currentTimeMillis();
        if (now < lastMillis) {
            throw new IllegalStateException("clock moved back by " + (lastMillis - now) + " ms");
        }
        if (now == lastMillis) {
            sequence = (sequence + 1) & SEQ_MASK;
            if (sequence == 0) {
                now = waitNext(lastMillis);
            }
        } else {
            sequence = 0;
        }
        lastMillis = now;
        return ((now - EPOCH) << TIME_SHIFT)
                | (roomId << ROOM_SHIFT)
                | (workerId << WORKER_SHIFT)
                | sequence;
    }

    private long waitNext(long floor) {
        long now = System.currentTimeMillis();
        while (now <= floor) {
            Thread.onSpinWait();
            now = System.currentTimeMillis();
        }
        return now;
    }

    public static void main(String[] args) {
        FlakeIds ids = new FlakeIds(2, 6);
        for (int i = 0; i < 8; i++) {
            System.out.println(ids.next());
        }
    }
}
```

`main` 在同一毫秒里连续取 8 个号，它们应当递增，低 12 位是 0 到 7。把纪元改掉之后，已经发出去的号不能再用新纪元解释，所以纪元属于协议的一部分，上线后不要动。

### 工程落地经验

#### 现成工具

业务项目通常不手写移位，而是用成熟工具里的发号器。自己写时最容易错的是机器号重复和时钟回拨，这两处要用部署系统管起来，而不是写在代码常量里。

#### 启动时绑定机器号

进程起来时确定机房和机器，再创建一个发号器。下面用主机名的哈希映射到 0 到 31，只适合机器很少、并且接受极端情况下撞号的演示。正式环境应使用配置中心或调度系统分配的编号。

```java
import jakarta.annotation.PostConstruct;
import java.net.InetAddress;

public class IdService {
    private int worker = 1;
    private final int room = 3;
    private FlakeIds flake;

    @PostConstruct
    public void init() {
        try {
            String host = InetAddress.getLocalHost().getHostName();
            worker = Math.floorMod(host.hashCode(), 32);
        } catch (Exception ex) {
            worker = 1;
        }
        flake = new FlakeIds(room, worker);
    }

    public synchronized long allocate() {
        return flake.next();
    }
}
```

多实例部署在同一台宿主机、又拿到相同主机名时，哈希会撞车。容器里要用平台注入的序号，例如有状态副本的编号，而不是主机名。

### 优缺点

#### 优点

- 时间在高位、序列在低位，整体趋势递增，对聚簇索引友好
- 不依赖数据库。发号只做位运算，延迟稳定
- 各段宽度可以按业务改。机器少就把机器位让给序列，换取单机更高的每毫秒吞吐

#### 缺点

- 依赖本机时钟。时钟往回跳会拒绝发号，或者在处理不当的实现里产生重复
- 多机之间不是严格单调。只要每台自己递增，就能满足大多数主键场景

#### 其它补充

回拨不能只靠抛异常。常见补救有：发号时记住上次时间，回拨幅度很小时自旋等到追上；幅度大时切换到备用机器号，或暂停这一小段序列。也有开源发号服务把号段缓存在本地，时钟问题交给中心节点统一处理。自研时先把“回拨就拒绝”做正确，再谈平滑扛过回拨。

号一旦对外公布，就不要在同一业务里更换位布局。解析监控、对账和分表路由都可能已经依赖其中的时间段。
