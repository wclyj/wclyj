# 声明式事务：注解在，数据却可能已经写进去了

Spring 把 JTA、JDBC、Hibernate、JPA 的事务收成同一套编程模型。声明式事务加上 Boot 的自动配置之后，很多项目只在方法上放一个 `@Transactional`，就认为一段数据库操作具备了原子性。

业务同学大多知道：几条写操作要同成功、同失败，就该放进事务。真正落地时却常常停在“方法上有注解”。注解有没有织进去、异常能不能让它回滚、一段流程里的子步骤该不该独立提交，这三件事对不上时，主路径测试往往是绿的。压力上来、异常变多之后，才会出现一批对不上的数据，只能人工修。

一个能扛异常的系统和一个“功能跑通”的系统，差别经常就在这些分支上。事务怎么划，也不全是框架问题，它跟着产品上“失败了要不要留下主单”的规则走。下面只处理技术侧最容易写错的三类，避免偶发的不一致。

## 注解写了，事务却没包住目标方法

先准备一张会员卡表，只有主键和昵称：

```java
@Entity
public class MemberCard {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long cardId;
    private String nickname;

    public MemberCard() {}

    public MemberCard(String nickname) {
        this.nickname = nickname;
    }

    public String getNickname() {
        return nickname;
    }
}
```

仓储用 Spring Data JPA，按昵称把行查回来，用来在接口末尾数“到底插进去几条”：

```java
public interface MemberRepository extends JpaRepository<MemberCard, Long> {
    List<MemberCard> findByNickname(String nickname);
}
```

### 私有方法上的注解不会被代理到

入口 `openBrokenPrivate` 是 public，真正写库的 `persistPrivate` 是 private，注解打在后者上。昵称里带 `sample` 就抛运行时异常，作者期望整次插入回滚。

```java
@Service
public class MemberService {
    @Autowired
    private MemberRepository memberRepository;

    public int openBrokenPrivate(String nickname) {
        try {
            persistPrivate(new MemberCard(nickname));
        } catch (RuntimeException ex) {
            log.error("open card rejected: {}", ex.getMessage());
        }
        return memberRepository.findByNickname(nickname).size();
    }

    @Transactional
    private void persistPrivate(MemberCard card) {
        memberRepository.save(card);
        if (card.getNickname().contains("sample")) {
            throw new RuntimeException("nickname rejected");
        }
    }
}
```

控制器只是把查询参数递进去。用 `sample-01` 调几次，库里仍能看见这些昵称，刷新之后行数还在涨。

原则一：默认的动态代理只增强 public 方法。Spring 用代理包住 Bean，在方法进入和退出时开关事务。private 方法对子类不可见，CGLIB 生成的子类覆盖不了它，代理就不会在那里开事务。`persistPrivate` 上的注解等于注释。真正发生的写，是仓储 `save` 自己那一层极短的事务：插入已经提交，后面的异常来不及撤销它。

除非改成 AspectJ 那种编译期或加载期织入，否则不要把 `@Transactional` 放在 private、protected 或 final 方法上。

### 同类自调用同样绕过代理

把 `persistPrivate` 改成 public，看起来满足了原则一。再加一个入口，在类内部用 `this` 去调它：

```java
public int openBrokenSelf(String nickname) {
    try {
        this.persistPublic(new MemberCard(nickname));
    } catch (RuntimeException ex) {
        log.error("open card rejected: {}", ex.getMessage());
    }
    return memberRepository.findByNickname(nickname).size();
}

@Transactional
public void persistPublic(MemberCard card) {
    memberRepository.save(card);
    if (card.getNickname().contains("sample")) {
        throw new RuntimeException("nickname rejected");
    }
}
```

用非法昵称再测，行还是留下来了。

原则二：必须走代理对象，增强才存在。`this.persistPublic(...)` 是一次普通的虚方法调用，编译器直接跳进当前实例，不会经过容器交给你的那个子类代理。代理对象上才挂着事务拦截器。

调试时可以给自己注入一份代理。容器注入的是 CGLIB 子类，断点里能看见类名带 `EnhancerBySpringCGLIB`：

```java
@Autowired
private MemberService self;

public int openViaProxy(String nickname) {
    try {
        self.persistPublic(new MemberCard(nickname));
    } catch (RuntimeException ex) {
        log.error("open card rejected: {}", ex.getMessage());
    }
    return memberRepository.findByNickname(nickname).size();
}
```

这样事务会生效，非法昵称插不进去。但服务自己注入自己，读代码的人很难看出这是为了绕过 `this`。更干净的做法是让控制器直接调用已经公开、已经加了注解的方法，异常也在边界上处理：

```java
@GetMapping("/cards/open")
public int open(@RequestParam String nickname) {
    try {
        memberService.persistPublic(new MemberCard(nickname));
    } catch (RuntimeException ex) {
        log.error("open card rejected: {}", ex.getMessage());
    }
    return memberService.countByNickname(nickname);
}
```

三种调用可以放在一起记：

- 类内部 `this`：永远不经过代理，方法上的事务注解不参与。
- 注入自身再调用：拿到的是代理，拦截器会跑。
- 别的 Bean（控制器、另一个服务）调用：同样走代理。

CGLIB 靠生成子类来拦截，所以 private 它看不见，final 它覆盖不了。JDK 动态代理则要求接口。`this` 指向目标对象本身，容器没有办法把 `this` 换成代理。

开发期把 JPA 的事务日志开到 DEBUG，能立刻分辨事务开在哪一层：

```properties
logging.level.org.springframework.orm.jpa = DEBUG
```

自调用时，日志里的事务名会落在仓储的 `save` 上，传播行为仍是 REQUIRED，但边界只包住这一次插入。从控制器打到 `persistPublic` 时，事务名会变成 `MemberService.persistPublic`。看见名字停在仓库实现类，就说明业务方法没有被增强。

有人会想：那给 `openBrokenSelf` 也加上注解，控制器只调这一个 public 方法，不就同时满足两条原则了？可以试，但下一类坑马上出现——事务开了，异常仍可能提交。

## 事务开了，也不代表会回滚

代理上的事务拦截器可以理解成包住目标方法的一段 try/catch：正常返回就提交；抛出的异常如果符合回滚规则，就标记回滚再把异常抛出去。简化之后是：

```text
开启事务
try {
    调用目标方法
    提交
} catch (Throwable error) {
    按规则决定回滚或继续提交
    把 error 再抛出
} finally {
    清掉线程上的事务上下文
}
```

规则里有两条最容易被忽略。

第一，异常必须飞出带注解的方法。拦截器只有在自己的 catch 里看见异常，才会走回滚。方法内部把异常吃掉，拦截器看到的是正常返回。

第二，默认只在 `RuntimeException` 和 `Error` 上回滚。受检异常被当成业务结果的一种，类似“另一种返回值”：调用方声明了它，就说明这条路径仍可能把资源收尾并提交。`Error` 和运行时异常则被视为没预料到的失败。这和编程式 `TransactionTemplate` 略有差别，模板对未声明的受检异常也会回滚；声明式事务则要求你用 `rollbackFor` 明确把受检异常算进去。

### 异常被吃掉

```java
@Transactional
public void openSwallow(String nickname) {
    try {
        memberRepository.save(new MemberCard(nickname));
        throw new RuntimeException("nickname rejected");
    } catch (RuntimeException rejected) {
        log.error("open card failed", rejected);
    }
}
```

运行时异常确实出现了，但没有离开 `openSwallow`。拦截器认为方法成功，事务提交，行留在表里。

若业务必须自己记日志、自己翻译错误，可以在捕获之后把当前事务标成只回滚：

```java
@Transactional
public void openMarkRollback(String nickname) {
    try {
        memberRepository.save(new MemberCard(nickname));
        throw new RuntimeException("nickname rejected");
    } catch (RuntimeException rejected) {
        log.error("open card failed", rejected);
        TransactionStatus status = TransactionAspectSupport.currentTransactionStatus();
        status.setRollbackOnly();
    }
}
```

DEBUG 日志里会先出现“事务代码要求回滚”这类字样，然后才是真正的 rollback。不要只看有没有 catch，要看标记有没有打上。

### 抛出的是受检异常

注册主档之后还要读一份本地合同。文件不存在时希望数据库也撤销。方法没有 catch，可是 `IOException` 是受检异常，默认规则不回滚。

```java
@Transactional
public void openThenRead(String nickname) throws IOException {
    memberRepository.save(new MemberCard(nickname));
    readContract();
}

private void readContract() throws IOException {
    Files.readAllLines(Path.of("missing-contract.txt"));
}
```

调用后表里仍有这条会员。修复是在注解上写明：碰到 `Exception` 就回滚。

```java
@Transactional(rollbackFor = IOException.class)
public void openThenRead(String nickname) throws IOException {
    memberRepository.save(new MemberCard(nickname));
    readContract();
}
```

再次调用，日志里能看到事务回滚，而不是提交。若只有少数受检异常代表“可以提交的业务拒绝”，用 `noRollbackFor` 把它们排除，其余仍回滚。不要复制默认行为却又口头约定“所有异常都要撤”。

这个例子里，数据库写和文件读被放进同一个一致性要求：文件失败，库也不能留。如果两次数据库写本来就应该各自提交，那不是 `rollbackFor` 能解决的，要改传播行为。

## 传播行为要和“失败是否拖垮主单”一致

产品规则：开通主卡时顺带写一张附属联系档。附属档失败只记日志，主卡必须留下。实现上却把两段都放进默认传播。

```java
@Transactional
public void openWithSideBroken(MemberCard card) {
    saveMain(card);
    sideProfileService.saveSideBroken(card);
}

private void saveMain(MemberCard card) {
    memberRepository.save(card);
    log.info("main card stored");
}
```

附属服务同样加了 `@Transactional`，写完就抛运行时异常：

```java
@Service
public class SideProfileService {
    @Autowired
    private MemberRepository memberRepository;

    @Transactional
    public void saveSideBroken(MemberCard card) {
        log.info("side profile begin");
        memberRepository.save(new MemberCard(card.getNickname() + "#side"));
        throw new RuntimeException("profile blocked");
    }
}
```

控制器捕获异常再去数行。日志里能看到整段事务回滚，接口打印的原因是 `profile blocked`。这并不意外：运行时异常穿出了外层的 `openWithSideBroken`，外层注解的默认规则就是回滚。主卡和附属档根本在同一个事务里。

下一步通常是把附属调用包进 catch，让外层不再看到异常：

```java
@Transactional
public void openWithSideCaught(MemberCard card) {
    saveMain(card);
    try {
        sideProfileService.saveSideBroken(card);
    } catch (RuntimeException ex) {
        log.error("side profile skipped: {}", ex.getMessage());
    }
}
```

结果更反直觉。DEBUG 顺序大致是：

1. 外层按 REQUIRED 新建事务，事务名是 `openWithSideCaught`。
2. 进入附属方法时，发现线程上已经绑定了 `EntityManager`，于是加入当前事务，而不是再开一个。
3. 附属方法抛运行时异常，参与者把“当前事务只能回滚”写上。
4. 外层 catch 住，打印 side profile skipped，方法正常返回。
5. 拦截器尝试提交，发现标记已是 rollback-only，提交失败。
6. 控制器收到 `UnexpectedRollbackException`。异常文本的意思是：事务被静默地回滚了，因为早先已经被标成不能提交。

所谓静默，是指 `openWithSideCaught` 自己没有再抛业务异常，失败发生在框架替你提交的那一下。外层捕获只是藏住了附属异常，藏不住已经写在共享事务上的回滚标记。默认传播 `REQUIRED` 的含义就是：有事务就加入，没有才新建。附属方法上的注解并没有把它拆出去。

要让附属档单独成败，传播得改成 `REQUIRES_NEW`：执行到该方法时挂起外层事务，新开一个事务；这个新事务回滚，不影响被挂起的外层。

```java
@Transactional(propagation = Propagation.REQUIRES_NEW) // 挂起外层
public void saveSideIsolated(MemberCard mainCard) {
    log.info("side profile begin");
    memberRepository.save(new MemberCard(mainCard.getNickname() + "#side"));
    throw new RuntimeException("profile blocked");
}
```

外层仍要捕获，否则附属异常会在内层事务回滚之后继续往外抛，外层自己的规则仍会把自己回滚掉。

```java
@Transactional
public void openWithSideIsolated(MemberCard card) {
    saveMain(card);
    try {
        sideProfileService.saveSideIsolated(card);
    } catch (RuntimeException ex) {
        log.error("side profile skipped: {}", ex.getMessage());
    }
}
```

关键日志变成另一条顺序：

1. 为 `openWithSideIsolated` 新建外层事务。
2. 主卡写入完成。
3. 挂起外层，为 `saveSideIsolated` 新建内层事务。
4. 内层回滚。
5. 恢复刚才挂起的外层事务。
6. 外层记下附属失败。
7. 外层提交。控制器不再看到 `UnexpectedRollbackException`。

按昵称计数，主卡那一条还在，带 `#side` 后缀的那一条不在。这才符合“附属失败不影响主单”。

`REQUIRES_NEW` 会真正占用第二条数据库连接，因为外层事务还没结束，不能在同一条连接上再开一个独立事务。连接池如果很小，这种嵌套会放大借用等待。不需要独立提交时，不要为了“看起来更安全”到处加 `REQUIRES_NEW`。

## 沿着一次非法昵称看连接

用 `sample-01` 走 `openBrokenPrivate`，时间顺序是这样的：

1. 控制器进入没有注解的 public 入口，线程上还没有业务事务。
2. `this.persistPrivate` 直接进目标对象，代理不参与。`save` 触发仓库自己的 REQUIRED：发现没有事务就新建，插入执行，仓库方法返回时这条短事务已经提交。
3. 回到私有方法的下一行，昵称检查抛出运行时异常。此时业务事务已经不在了，异常没有东西可回滚。
4. 入口 catch 住，再按昵称查询，能看见刚提交的行，返回值大于 0。

同一昵称改由控制器直接调用 `persistPublic`：

1. 拦截器在进方法之前向事务管理器申请事务，连接上的自动提交被关掉。
2. `save` 发现线程上已经有事务，只把插入加入，并不自己提交。
3. 运行时异常飞出方法。拦截器认定它属于默认回滚类型，连接执行 rollback，这条插入对其他事务不可见。
4. 控制器 catch 之后再计数，得到 0。

`openBrokenSelf` 即使把目标方法改成 public，顺序仍接近私有方法：调用点是 `this`，拦截器没有包住这次进入，提交发生在仓库 `save` 返回的时候。注入 `self` 之后，调用点落到代理的方法表上，顺序才和控制器直接调用一致。看 DEBUG 日志时，以事务名上的全限定方法为准，不要以注解写在源码哪一行为准。名字停在仓库实现类，就说明业务方法被绕过去了。

## 捕获异常为什么挡不住回滚标记

`REQUIRED` 加入外层事务时，内层没有自己的提交点。内层抛运行时异常，内层拦截器先把共享事务标成只能回滚，再把异常抛出去。外层 catch 住，只是让外层方法可以正常返回，连接上的标记还在。外层拦截器随后走提交分支，管理器看到标记，拒绝提交，抛出 `UnexpectedRollbackException`。日志里会先出现“开始提交”，紧接着才是失败。不要把开始提交理解成数据已经落袋。

`REQUIRES_NEW` 把标记隔离开：内层的 rollback-only 写在内层那条连接上，回滚也只作用于它。外层从挂起里恢复时，自己的标记仍允许提交。前提是内层异常不要再穿透外层的注解方法。穿透了，外层会按自己的规则再决定一次，主卡仍可能被撤掉。

附属方法里如果还有第二条 insert，隔离的是“这一整个方法”，不是某一条语句。同一个新事务里的两条写仍然同进同退。希望它们互不影响，就要再拆方法、再指定传播，而不是在一个方法里分多次 catch。

`REQUIRES_NEW` 挂起外层时，外层连接不能借去给内层用，池要再分一条连接。池如果只有两三条，主流程再加一两个独立子事务，就会在借连接上排队，甚至先于业务异常超时。传播方式和连接池上限要一起看。

## 评审时按这个清单扫一遍

- 注解方法是不是 public，是不是 final。final 方法 CGLIB 覆盖不了。
- 调用点是不是另一个 Spring Bean。重点搜 `this.` 后面紧跟着的、带注解的方法名。
- 方法体里有没有把异常一网打尽的 catch。有的话，要么重抛，要么 `setRollbackOnly`。
- 签名上有没有 `throws` 受检异常。有的话，核对 `rollbackFor` 是否包含它。
- 同一用例里的两个注解方法是不是都用默认传播。产品若要求子步骤失败不影响主单，子方法要用 `REQUIRES_NEW`，外层还得把异常接住。补偿式的后续任务（发通知、写审计）若不能跟主单绑在一个事务里，同样不要放进这个方法。
- 事务方法里有没有远程调用或长睡眠。有的话，连接占用时间按整段算，池的上限跟着改。否则事务配置是对的，系统却在借连接上失败。

单元测试不要只断言“正常昵称能查到”。至少再加三条：非法昵称计数为 0、缺失合同文件时计数为 0、附属档抛错时主卡计数为 1 且带后缀的那条不存在。这三条分别钉住自调用、受检异常和传播。重构时一旦有人把调用改回 `this`，或者把 `REQUIRES_NEW` 删掉，测试会先红。

只读查询若误加了事务，默认仍会占用连接直到方法结束。报表类接口如果在事务里再去调远程服务，会把连接占用拉长，池被读请求耗尽，写请求跟着超时。读方法可以用只读标记，并避免在其中做远程调用。

## 回顾

- 没生效：注解所在方法要是 public，并且要从别的 Bean 或注入的代理来调用。类内部的 `this` 自调用、private 方法，默认动态代理都碰不到。用事务日志核对事务名是业务方法还是仓库的 `save`。
- 生效了却没回滚：异常要飞出注解方法；默认只处理运行时异常和 `Error`。自己 catch 了就调用 `setRollbackOnly`，或者不要 catch。希望受检异常也回滚时，设置 `rollbackFor`；个别业务异常需要提交时，用 `noRollbackFor`。
- 一段流程里有多段写操作，而且失败边界不同：检查 `propagation`。默认 REQUIRED 是加入当前事务，内层把事务标成 rollback-only 之后，外层即使捕获异常也提交不了。附属逻辑要独立回滚时用 REQUIRES_NEW，同时仍要挡住异常，别让它冲出外层。

这些分支不在快乐路径上，代码评审时应专门看一遍：谁在调用带注解的方法，异常出不出得去，传播是不是把两段写绑在一起了。单元测试至少覆盖“非法参数”“受检异常”“子步骤失败但主步骤要保留”这几种，重构调用关系时才不会把事务边界改丢。

## 换 MyBatis 时还要看什么

上面为了短，数据访问用的是 Spring Data JPA。国内项目更多是 MyBatis。声明式事务并不读取 SQL 写法，它只包住 Spring 管理的那个 Bean 方法，所以三类坑一个都不会少：Mapper 上的注解同样救不了 Service 里的自调用；在 Service 里吞掉运行时异常，MyBatis 的 insert 一样会提交；子服务默认加入外层事务，捕获之后仍可能在提交阶段得到 `UnexpectedRollbackException`。

可以把 `MemberRepository.save` 换成 Mapper 的 `insert`，把日志级别调到 MyBatis 的 JDBC 或 Spring 的 `DataSourceTransactionManager`。自调用时，你看不到围绕 `persistPublic` 的事务边界，只能看到连接被借出、执行 insert、然后提交。从控制器进入 public 方法时，事务名会回到 Service。用这个差别来确认，比肉眼看注解可靠。

如果一定要让 private 方法参与事务，动态代理做不到，需要在编译期用 AspectJ 把增强织进字节码。官方文档有一节专门讲 `@Transactional` 与 AspectJ 的配合。和 Lombok 一起用时要小心：两者都在编译期改语法树，处理器顺序不对时，可能出现注解织到了 Lombok 还没生成的方法上，或者生成的 getter 没被织入。启用之前先用一个 private 方法的回滚测试固定住行为，再谈推广。
