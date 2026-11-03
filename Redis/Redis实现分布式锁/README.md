# Redis实现分布式锁

## 前言

多台机器同时改同一行库存时，单进程里的 `synchronized` 管不到别的进程。常见的跨进程互斥有三条路：数据库里的乐观版本号或唯一约束、Redis 上的租约锁、ZooKeeper 的临时顺序节点。下面只展开 Redis 这一条，并把写错的写法对照出来。

## 分布式锁满足的条件

一把能上生产的锁，至少要同时满足：

- 互斥：同一时刻只有一个持有者
- 不会永久占住：持有者进程若崩溃、没来得及删除键，其他调用方之后仍能拿到锁
- 故障下仍可加锁解锁：集群里多数节点还活着时，客户端应能完成获取和释放。单机部署谈不上这条，多节点才需要认真设计
- 只能由持有者释放：一个客户端不能删掉别人刚写入的键

## 加锁代码

### 正确示例

获取锁应落成一条命令，同时写上「仅当不存在」和过期时间。Jedis 3 的参数对象写法如下，语义与旧版五个参数的 `set` 相同：

```java
public final class LeaseLock {
    private LeaseLock() {}

    public static boolean acquire(Jedis redis, String key, String owner, long ttlMillis) {
        SetParams params = new SetParams().nx().px(ttlMillis);
        String reply = redis.set(key, owner, params);
        return "OK".equals(reply);
    }
}
```

命令行等价形式：

```bash
redis-cli SET lock:stock:42 "$OWNER" NX PX 8000
```

五个信息各自的职责：

- 键是锁的名字，必须唯一对应一个资源，例如某张单据或某一行库存
- 值是持有者标识，不能只写一个固定的 `"1"`。释放时要证明「这把锁是我加的」。标识可以用 `UUID.randomUUID()` 生成
- `NX` 表示仅在键不存在时写入。键已经在，命令返回空，调用方加锁失败
- `PX` 表示过期时间用毫秒。`EX` 则是秒
- 最后一个数字是租约长度。超时后键消失，锁自动放开

于是只有两种结果。键不存在：写入成功，带上持有者和租约。键已存在：什么都不改，调用方返回失败。

对照前面的条件：`NX` 保证互斥；租约保证持有者崩溃后锁会自己消失；值里的持有者标识保证释放时能核对身份。单机 Redis 没有「多数节点」可言，第三条先不讨论。

### 错误示例-1

把「不存在才写入」和「设置过期」拆成两条命令：

```java
public static void acquireBroken(Jedis redis, String key, String owner, int ttlSeconds) {
    Long created = redis.setnx(key, owner);
    if (created != null && created == 1L) {
        // 进程若在这条命令之前退出，键永远留在库里
        redis.expire(key, ttlSeconds);
    }
}
```

`SETNX` 与 `EXPIRE` 各自是一次往返。第一条成功后进程被杀掉、网络被掐断，键就没有过期时间，后面的请求永远进不来。旧版客户端没有带 `NX` 和 `PX` 的 `SET`，才逼得有人写成两步。现在应改成一条 `SET`。

### 错误示例-2

另一种写法把「绝对到期时刻」塞进值里，过期了就用 `GETSET` 抢锁：

```java
public static boolean acquireByTimestamp(Jedis redis, String key, int ttlMillis) {
    long deadline = System.currentTimeMillis() + ttlMillis;
    String token = Long.toString(deadline);
    if (redis.setnx(key, token) == 1L) {
        return true;
    }
    String current = redis.get(key);
    if (current != null && Long.parseLong(current) < System.currentTimeMillis()) {
        String previous = redis.getSet(key, token);
        return previous != null && previous.equals(current);
    }
    return false;
}
```

看起来只有一个线程能在 `GETSET` 里看到旧值，从而「赢下」这把锁。问题有三处：

1. 到期时刻由各台机器的本地时钟生成，时钟一旦偏差，一台机器会认为锁已失效，另一台仍认为有效
2. 多个客户端同时 `GETSET`，最后写入的那个时间戳会盖住真正持有者刚刚写上的租约，锁的到期被别人改短或改长
3. 值里没有持有者身份，谁都能删

## 解锁代码

### 正确示例

释放必须是「先比对持有者，再删除」，而且两步要原子。Lua 在服务器里被当成一条命令执行，脚本跑完之前不会插入其他命令：

```java
public static boolean release(Jedis redis, String key, String owner) {
    String script = "local holder = redis.call('GET', KEYS[1])\n"
            + "if holder == ARGV[1] then\n"
            + "  return redis.call('DEL', KEYS[1])\n"
            + "end\n"
            + "return 0";
    Object reply = redis.eval(script,
            Collections.singletonList(key),
            Collections.singletonList(owner));
    return Long.valueOf(1L).equals(reply);
}
```

`KEYS[1]` 是锁的键，`ARGV[1]` 是当初写入的持有者。值相等才 `DEL`。不相等说明锁已经不属于这次调用，返回 0。不要在客户端先 `GET` 再 `DEL`，那两步之间锁可能已经过期并被别人重新持有。

### 错误示例1

直接删除，不核对值：

```java
public static void releaseBroken(Jedis redis, String key) {
    redis.del(key);
}
```

任何拿到这个键名的客户端都能解开别人的锁。键名一旦泄漏，互斥就形同虚设。

### 错误示例2

比对和删除拆开，中间留下窗口：

```java
public static void releaseSplit(Jedis redis, String key, String owner) {
    if (owner.equals(redis.get(key))) {
        // 走到这里时租约可能已经到期，键被另一个调用方重新写入
        redis.del(key);
    }
}
```

时间线可以这样发生：调用方 A 持有锁，租约剩很少；A 读到自己的标识，还没执行删除，键先过期；调用方 B 加锁成功；A 继续执行删除，把 B 的键删掉。B 以为自己仍独占资源，A 已经把锁放掉了。

# 总结

单机 Redis 上的锁，获取用带 `NX` 和过期时间的 `SET`，释放用一段比对后再删除的 Lua。四条要求里，互斥、租约、持有者核对，这三条都能在单机上做到。多数节点仍能加锁，属于多副本算法要单独解决的问题。

主从复制时还有一个缺口：锁写在主节点上，主节点还没来得及把键同步出去就宕机，从节点升主后锁消失，两个客户端会同时认为自己持有。多节点部署不要只用上面这段单机脚本。可以评估带看门狗续期的客户端实现，也可以把租约和业务幂等一起做：即使锁偶然失效，重复执行也不会把库存减成负数。续期看门狗要放在持有者进程里，进程若真的卡死，续期也会停，租约仍能兜底。
