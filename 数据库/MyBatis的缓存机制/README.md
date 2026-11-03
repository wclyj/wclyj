# MyBatis 缓存机制

## 前言

ORM 普遍会做查询缓存，用来少打数据库。MyBatis 有两级：跟一次会话绑在一起的本地缓存，以及按映射文件命名空间共享的二级缓存。二级缓存的存储可以换成自己的实现。

和缓存相关的类型集中在 `org.apache.ibatis.cache`。接口 `Cache` 的基础实现是 `PerpetualCache`，内部就是一个 `HashMap`，没有容量上限，也不会过期。调用链可以看成：

`DefaultSqlSession` 持有一个 `Executor`。普通执行器的父类 `BaseExecutor` 里有一个 `PerpetualCache`，字段名是 `localCache`。这就是一级缓存的存放点。

真正用到的往往不是裸的 `PerpetualCache`，而是套了好几层装饰器：按 LRU 或 FIFO 淘汰、打日志、按间隔清空、把值序列化成副本、加同步、把写入推迟到事务提交。装饰器再多，最里面仍是那张哈希表。开启二级缓存时，最外层还会包一个 `CachingExecutor`。调试时顺着这个装饰器看 `delegate`，就能回到真正执行 SQL 的那个执行器。

这些实现可以分成三组。基本存储只有 `PerpetualCache`。淘汰策略是 `LruCache`、`FifoCache`，以及靠软引用、弱引用把淘汰交给垃圾回收器的两种。其余都是装饰器，给同一套 `getObject` / `putObject` 加行为。

## 一级缓存（本地缓存）

这一级绑在单次 `SqlSession` 上，默认就会启用，映射文件里不用再声明。有人把它叫本地缓存，意思是别的会话看不见。

`DefaultSqlSession` 里和这次会话相关的可变状态主要在执行器上。`Configuration` 是全局共享的，不能拿来放「只有这次会话能看见」的结果。所以缓存放在 `BaseExecutor` 的构造过程里创建的那个 `PerpetualCache`。`SimpleExecutor`、`ReuseExecutor`、`BatchExecutor` 都继承这条路径。同一个会话里，相同的查询会直接拿内存里的结果，不再把 SQL 发到数据库。换一个会话，哪怕 Mapper 方法、参数完全一样，也看不到对方的一级缓存。

一次会话里，业务代码可能用同样的条件查好几次。每次都访问数据库既慢又加重负载，而极短时间内结果通常没变。MyBatis 于是在会话上记一份简单缓存：查到结果就放进去；下次若判定是同一条查询，就把保存的对象交回去。

查询时先根据语句 id、参数、分页用的 `RowBounds` 组装 `CacheKey`，再拿这个键去 `localCache` 查找。命中就返回。未命中则访问数据库，把结果按同一把键放进缓存，然后返回。这个读写发生在 `BaseExecutor.query`：先 `getObject`，没有再进入 `queryFromDatabase`。

本地缓存能活多久，由会话上的几个动作决定：

1. `openSession` 会新建 `SqlSession`、新建执行器、新建空的 `PerpetualCache`。会话结束，这三个对象一起被丢掉
2. `close()` 之后，这份额缓存不可再用
3. `clearCache()` 只清条目，`PerpetualCache` 对象还在，后续查询可以继续往里放
4. 同一次会话里只要执行了 `insert`、`update` 或 `delete`，`BaseExecutor.update` 会先无条件 `clearLocalCache()`，再真正写数据库。不需要等到提交。提交影响的是事务边界，不是这张本地表还在不在

因此「不提交就不会清一级缓存」是把两级缓存记混了。更新方法一被调用，本地缓存就已经空了。若事务最终回滚，数据库里的修改消失，但本地缓存并不会自动把旧结果填回来，下次查询会重新读库，在同一个未提交的连接上看到当前事务能看见的数据。

工作流程压缩成四步：用语句 id、参数和 `RowBounds` 做键；按键查找；命中则返回；未命中则查库、写入、再返回。

下面用订单查询看三个现象。判断是否命中，不要看返回值相不相等，而看日志里 SQL 发了几次。需要把日志实现设成标准输出，配置项是 `logImpl` 指向 `StdOutImpl`。

同一会话里连续查两次，只应出现一条 SQL：

```java
SqlSession session = factory.openSession();
OrderMapper orders = session.getMapper(OrderMapper.class);
orders.selectById(501L);
orders.selectById(501L);
```

第二次没有访问数据库，说明命中的是本地缓存。注意返回的往往是同一个对象引用。调用方若改了这个对象的字段，等于改了缓存里的内容，随后的读取会看到被改过的值。

同一会话里，更新会把本地缓存清掉，后面的查询会再访问数据库：

```java
OrderMapper orders = session.getMapper(OrderMapper.class);
orders.selectById(501L);
Order patch = new Order();
patch.setId(501L);
patch.setTitle("改后备注");
orders.updateTitle(patch);
orders.selectById(501L);
```

第二次 `selectById` 会再次打印 SQL。清空发生在 `update` 里面，不是发生在 `commit` 里面。

另一个会话更新并提交之后，当前会话如果之前缓存过同一行，仍可能交出旧对象。一级缓存不跨会话，也看不见别人的提交：

```java
SqlSession reader = factory.openSession();
SqlSession writer = factory.openSession();
OrderMapper readSide = reader.getMapper(OrderMapper.class);
OrderMapper writeSide = writer.getMapper(OrderMapper.class);
readSide.selectById(501L);
Order patch = new Order();
patch.setId(501L);
patch.setTitle("另一会话已提交");
writeSide.updateTitle(patch);
writer.commit();
readSide.selectById(501L);
```

最后一次查询若不再打印 SQL，页面上就是脏数据：库里的标题已经变了，`reader` 仍抱着第一次放进本地缓存的对象。要避免它，可以缩短会话，让一次请求对应一次 `SqlSession`；或者把本地缓存级别调成 `STATEMENT`。

`localCacheScope` 有两个取值。`SESSION` 是默认，整个会话内相同查询不重复打库。`STATEMENT` 表示每条语句结束就清本地缓存，用来躲开上面的脏读，代价是同一次请求里的重复查询也不再复用。一级缓存不能从配置里整个关掉，只能把范围缩到语句级。

## 二级缓存：

二级缓存用来补「会话之间不能共享」这个缺口。范围是映射文件的 namespace，通常一个 Mapper 接口一份。同一个接口上的查询可以被多个 `SqlSession` 共用，生命周期跟应用进程走，而不是跟某一次会话走。

查询顺序是：先看二级缓存，没有再看当前会话的一级缓存，再没有才打数据库。所以二级缓存一定包在会话执行器的外面，否则别的会话碰不到它。

MyBatis 用 `CachingExecutor` 做这层外壳。`cacheEnabled` 不是 `false` 时，创建执行器就会套上它。处理查询时，它先按 namespace 找那份 `Cache`；命中则直接返回。未命中则交给内部真正的执行器，那一层才会走到一级缓存和数据库。查到结果后先放进事务缓冲，提交成功才对其他会话可见。

打开的步骤有两步，缺一不可。

第一步，全局 `cacheEnabled` 保持开启。默认就是开启。关掉之后不会再创建 `CachingExecutor`。

第二步，在对应的映射文件上声明 `cache`。例如用 LRU、最多保留 1024 条、两分钟刷新一次、返回可修改的副本：

```xml
<cache eviction="LRU" flushInterval="120000" size="1024" readOnly="false"/>
```

声明之后，这个文件里的查询默认走缓存；插入、更新、删除默认清空这份缓存。若不写 `flushInterval`，就不会按时间刷新，只在写语句触发时清空。`size` 是缓存里保留的引用个数上限，默认 1024，不是字节数。`readOnly="false"` 时，取出的对象要经过序列化得到副本，调用方修改它不会影响缓存里的那份，因此实体必须可序列化。设成 `true` 则把同一个实例交给所有调用方，更快，但谁改了字段，其他线程都会看见。

淘汰策略有四种：

- `LRU`：丢掉最久没有被访问的条目，这是默认
- `FIFO`：按进入缓存的先后丢掉最早的那条
- `SOFT`：用软引用持有，内存紧的时候更容易被垃圾回收器收走
- `WEAK`：用弱引用，回收更积极，命中率通常更差

`flushInterval` 的单位是毫秒，必须是正整数。`size` 也必须是正整数，要按单条结果的体积估算，一条查询若返回大列表，1024 个引用可能已经很大。

二级缓存是事务性的。`SqlSession` 提交时，这次事务里产生的查询结果才真正进入共享缓存；回滚则丢掉尚未发布的那些条目。写语句若显式标了 `flushCache="false"`，提交时也不会因为这条写语句去清空缓存。

只开全局开关、映射文件里却没有 `cache`，会怎样？`CachingExecutor` 仍然会包住执行器，但这个 namespace 没有自己的 `Cache` 对象，查询不会命中二级缓存。有没有共享缓存，取决于文件里有没有 `cache` 或 `cache-ref`。

个别查询要求实时，可以在语句上关掉：

```xml
<select id="selectById" resultType="Order" useCache="false">
```

默认值是：查询 `useCache="true"`、`flushCache="false"`；插入、更新、删除 `flushCache="true"`。一般不用把默认值再写一遍。只有想改变默认时才写，例如某条更新只改了一张与本缓存无关的表，可以考虑 `flushCache="false"`，但要自己证明不会脏读。

有三个现象值得自己跑一遍。

事务不提交，其他会话看不见这次查询的二级缓存。`CachingExecutor` 把结果先放进 `TransactionalCache` 的待提交表，键暂时不进真正的 `PerpetualCache`。`commit` 时才 `flushPendingEntries`。这就是「查询已经执行，另一会话却仍然打到数据库」的原因。回滚会清空待提交表。

提交之后，换一个 `SqlSession`、用同一个 Mapper 的同一个方法、同一个参数，不应再出现 SQL。这说明二级缓存跨会话。

其他会话做了更新并提交，这份 namespace 的缓存会被刷掉，随后的查询重新打库。原因是写语句默认 `flushCache` 为真，`CachingExecutor.update` 里会按这个标志清空。清空的是整个 namespace 的缓存，不是只丢掉被改的那一行。所以一条无关紧要的更新也会让该文件里所有查询的缓存失效。

什么场景才值得开：

- 写很少、读很多，例如历史订单、字典、已经封账的报表。写一旦频繁，缓存刚放进去就被清空，命中率接近零，还多了序列化成本
- 一个 namespace 最好只碰一张表。两份映射都操作订单表时，在其中一份里更新只会刷新那一份的缓存，另一份仍可能交出旧数据

多个 namespace 若确实要共用同一份缓存，用 `cache-ref` 指向对方的 namespace：

```xml
<cache-ref namespace="app.dao.OrderMapper"/>
```

两边的读写于是操作同一块存储，任何一边的更新都会把共享缓存清掉。关联表一多，刷新会非常密，共享之后的命中率要重新评估，不要为了省事把整个库的映射都指到同一份缓存上。

除了内置实现，还可以把 `cache` 的 `type` 换成自己的类，或换成官方维护的 Ehcache、Redis 适配。适配器实现同一个 `Cache` 接口即可。类必须提供一个接收字符串 id 的构造器，id 就是 namespace。需要的方法包括：返回这个 id、报告条目数、放入、取出、判断键是否存在、删除一个键、清空。可选属性写成 JavaBean 的 setter，在 XML 里用 `property` 传入，例如缓存文件路径。属性支持 `${...}`，值来自配置里的占位符。3.4.2 起，若实现 `InitializingObject`，全部属性设置完会再调用 `initialize`。

接口的包名是 `org.apache.ibatis.cache.Cache`。有的旧文写成 `org.mybatis.cache.Cache`，那个包并不存在。

自定义实现要注意线程安全，多个会话会同时调用它。如果底层用本地 `HashMap`，要自己加锁，或者包一层同步装饰。放进 Redis 时，键要带上 namespace，避免两个应用共用一个库时撞键；值要能序列化；还要给过期时间，免得映射文件改了旧条目还在。

缓存键不是 SQL 字符串本身。一级和二级都使用 `CacheKey`，里面依次放进语句 id、查询结果在结果集里的偏移和条数（也就是 `RowBounds`）、SQL 文本，以及每个绑定参数的值。少写任何一个，两条本来不同的查询就会共用一条结果。分页参数因此必须进键，否则第一页的列表会被第二页命中。

二级缓存取出对象时，若 `readOnly` 为假，装饰器会把值序列化再反序列化。实体没实现 `Serializable`，或者里面夹了不可序列化的连接、流、线程本地变量，提交阶段就会失败。嵌套的懒加载代理也不能直接进二级缓存，代理上挂着会话，会话关闭之后再反序列化没有意义。需要进共享缓存的查询，应把关联写成已经装好的普通对象，或者关掉这条语句的懒加载。

更新语句清空的是整个 namespace，所以「只改了一行」和「表结构迁移时跑了一条更新」对缓存的效果一样，都是全部失效。字典表和订单表不要写在同一个映射文件里，否则订单的每次状态变更都会把字典缓存打掉。反过来，只读字典可以单独一个 namespace，把 `flushInterval` 设成几分钟，容忍短暂的旧值。

两个会话都打开了二级缓存时，会话 A 查出的订单列表在提交之前不会进入共享区，会话 B 仍然访问数据库。A 提交之后，B 的下一次查询会先命中二级缓存。查询顺序是先二级、后一级，所以 B 不会被自己会话里更早放进去的旧一级缓存挡住。已经交回调用方的那个 Java 对象不会自动变新，页面若一直握着旧引用，要由调用方重新查询。

最后记住两级的分工。一级缓存保一次会话里的重复读取，默认存在，会话内的任何写都会把它清掉，它看不见其他会话的提交。二级缓存保多个会话之间的重复读取，必须在映射文件上声明，提交之后才发布，同一 namespace 的写会整表清空。页面要求读到刚提交的数据时，先缩短会话或把本地缓存降到语句级，再决定这张表配不配得上二级缓存。提交失败或回滚时，待写入共享区的条目会被丢掉，不会把半截事务的查询结果留在二级缓存里。
