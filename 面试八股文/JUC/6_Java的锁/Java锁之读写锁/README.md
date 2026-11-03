# 独占锁（写锁） / 共享锁（读锁） / 互斥锁

## 概念

独占锁：同一时刻只允许一个线程持有。`ReentrantLock` 和 `synchronized` 都是这种锁。

共享锁：同一时刻可以有多个线程一起持有，只要它们做的事情彼此不冲突。

`ReentrantReadWriteLock` 把两把锁捆在一起：读锁共享，写锁独占。

写的时候只能留一个线程；没有任何线程在写时，多个线程可以同时读。

## 为什么会有写锁和读锁

若缓存只用 `ReentrantLock`，读也要互斥。读本身不会把数据写坏，却被串行化，并发度被白白砍掉。读写锁把规则收成三条：

```text
多个线程同时读同一份缓存，可以并行。
只要有一个线程要写，其他线程就不能再读，也不能再写。
```

读-读：可以同时持有。

读-写：互相排斥。

写-写：互相排斥。

## 代码实现

先看不加锁的缓存。写到一半时，读线程已经进来，会读到还没放进去的空值，写线程之间也会交错。

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * 没有读写锁时，写操作会被别的线程插进来。
 */
class Shelf {
    private final Map<String, Object> slots = new HashMap<>();

    public void put(String key, Object value) {
        System.out.println(Thread.currentThread().getName() + " 开始放入 " + key);
        try {
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
        slots.put(key, value);
        System.out.println(Thread.currentThread().getName() + " 放入完成");
    }

    public void get(String key) {
        System.out.println(Thread.currentThread().getName() + " 开始查看 " + key);
        try {
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
        Object value = slots.get(key);
        System.out.println(Thread.currentThread().getName() + " 看到 " + value);
    }
}

public class ShelfRaceDemo {
    public static void main(String[] args) {
        Shelf shelf = new Shelf();
        for (int i = 1; i <= 4; i++) {
            final int code = i;
            new Thread(() -> shelf.put("k" + code, "v" + code), "w" + i).start();
        }
        for (int i = 1; i <= 4; i++) {
            final int code = i;
            new Thread(() -> shelf.get("k" + code), "r" + i).start();
        }
    }
}
```

四个写线程和四个读线程一起跑，可能出现：

```text
w1 开始放入 k1
w3 开始放入 k3
w2 开始放入 k2
r1 开始查看 k1
w4 开始放入 k4
r2 开始查看 k2
w1 放入完成
r1 看到 null
w3 放入完成
r3 开始查看 k3
```

“开始放入”还没配对“放入完成”，读线程已经打印 `null`。`HashMap` 本身也不是并发安全的，这种交错除了读到空值，还可能破坏表结构。

## 解决方法

写路径拿写锁，读路径拿读锁：

```java
private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
```

写的时候：

```java
rw.writeLock().lock();
// ... 修改 map ...
rw.writeLock().unlock();
```

读的时候：

```java
rw.readLock().lock();
// ... 读取 map ...
rw.readLock().unlock();
```

写锁是独占的，一个写者进入后，其他写者和读者都进不来。读锁是共享的，没有写者时多个读者可以叠在一起。`unlock` 放在 `finally`，避免异常把锁留在当前线程上。

完整代码：

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * 写独占、读共享。map 引用本身不再换，但表结构仍只在写锁里修改。
 */
class GuardedShelf {
    private final Map<String, Object> slots = new HashMap<>();
    private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();

    public void put(String key, Object value) {
        rw.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 开始放入 " + key);
            try {
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
            slots.put(key, value);
            System.out.println(Thread.currentThread().getName() + " 放入完成");
        } finally {
            rw.writeLock().unlock();
        }
    }

    public void get(String key) {
        rw.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 开始查看 " + key);
            try {
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
            Object value = slots.get(key);
            System.out.println(Thread.currentThread().getName() + " 看到 " + value);
        } finally {
            rw.readLock().unlock();
        }
    }

    public void clear() {
        rw.writeLock().lock();
        try {
            slots.clear();
        } finally {
            rw.writeLock().unlock();
        }
    }
}

public class ShelfRwDemo {
    public static void main(String[] args) {
        GuardedShelf shelf = new GuardedShelf();
        for (int i = 1; i <= 4; i++) {
            final int code = i;
            new Thread(() -> shelf.put("k" + code, "v" + code), "w" + i).start();
        }
        for (int i = 1; i <= 4; i++) {
            final int code = i;
            new Thread(() -> shelf.get("k" + code), "r" + i).start();
        }
    }
}
```

一种符合锁语义的输出：

```text
w1 开始放入 k1
w1 放入完成
w2 开始放入 k2
w2 放入完成
w3 开始放入 k3
w3 放入完成
w4 开始放入 k4
w4 放入完成
r2 开始查看 k2
r1 开始查看 k1
r4 开始查看 k4
r3 开始查看 k3
r2 看到 v2
r1 看到 v1
r4 看到 v4
r3 看到 v3
```

写操作一对一对出现，中间不会夹进别的“开始放入”或“开始查看”。四个读线程可以同时处于“开始查看”，因为读锁共享。若某个写线程还没结束，新的读者会停在 `readLock().lock()`，不会读到半成品。

注意：示例里用 `HashMap` 是安全的，只因为所有结构性修改都在写锁里，所有读都在读锁里。把锁拿掉之后，不能留着 `HashMap` 继续并发访问。
