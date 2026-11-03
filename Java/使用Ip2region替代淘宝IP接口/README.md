# 用离线地址库代替远程 IP 查询

## 远程接口不可用时，登录日志会被拖死

访问日志里要记一笔“这个地址大概在哪一座城市”。原先每次请求都去调一家公网 IP 查询接口。那家接口停过一段，调用方的读超时又很长，登录接口的耗时跟着抬高，最后只能把这段调用先摘掉。后来接口恢复了，却改成要带访问密钥，申请入口也找不到，远程查询就不适合继续当依赖。

更合适的做法是把地址库放在进程本地：给出 IP，直接在内存或本地文件里算出国家、地区、城市和运营商，不再为这一次查询发网络请求。下面用 ip2region 这套离线库说明怎么接、怎么选算法，以及打成 jar 之后为什么会找不到数据文件。

## 三种查法，差别在数据和文件谁更累

库带一份只有几 MB 的数据文件，查询实现分三档：

- 内存算法：启动时把整份数据读进堆，单次查询最快，代价是常驻内存。
- 二分算法：不把整库装进堆，按文件做二分，延迟通常低于 1 毫秒，但比内存算法慢。
- B 树算法：同样基于文件、不整库进堆，复杂度比二分更适合这种索引，一般比二分更快。

登录这种读多、可以接受首次稍慢的场景，用 B 树即可，再把查到的结果放进 Redis，避免同一 IP 反复翻文件。

## 依赖和数据文件放哪

Maven 使用 1.7 这一代工件：组织名 `org.lionsoul`，工件名 `ip2region`，版本号 `1.7.2`。更新的 xdb 格式是另一套 API，和下面的 `DbSearcher` 不是同一代，不要混用坐标和调用。

到数据目录把 `ip2region.db` 下载下来，放进资源目录 `src/main/resources/geo/`。不要把绝对路径写死在代码里，否则换机器就失效。

## 先在开发机上把查询跑通

工具类按 IP 返回去掉占位符之后的地区串。1.7 的查询器要配一个 `DbConfig`，再用文件路径构造 `DbSearcher`。三种算法对应三个方法：`btreeSearch`、`binarySearch`、`memorySearch`。内存算法那个常量在旧版本里的拼写是 `MEMORY_ALGORITYM`，调用时以库里的常量名为准。

这里固定走 B 树。开发阶段如果确认资源能展开成真实文件，可以直接用 `getResource` 的路径；下一节会处理 jar 里没有真实路径的问题。

```java
public final class RegionLookup {
    private RegionLookup() {}

    public static String locate(String ip) {
        if (!Util.isIpAddress(ip)) {
            return null;
        }
        URL resource = RegionLookup.class.getResource("/geo/ip2region.db");
        if (resource == null) {
            return null;
        }
        try {
            DbSearcher searcher = new DbSearcher(new DbConfig(), resource.getPath());
            DataBlock block = searcher.btreeSearch(ip);
            return tidy(block.getRegion());
        } catch (Exception ex) {
            log.warn("region lookup failed for {}", ip, ex);
            return null;
        }
    }

    static String tidy(String raw) {
        if (raw == null || raw.isEmpty()) {
            return null;
        }
        return raw.replace("|0", "").replace("0|", "");
    }
}
```

用一个公网地址做冒烟，例如 `114.114.114.114`。返回值是用竖线拼起来的若干段，空段在库里常写成 `0`，`tidy` 把这些占位去掉，日志里就只剩有意义的地名和运营商。

若要对比另外两种算法，把 `btreeSearch` 换成 `binarySearch` 或 `memorySearch` 即可，不要在每次查询里用反射再选一次方法。反射那层对这种固定选择没有收益，还把异常类型搅在一起。

查完写入 Redis，过期时间 12 小时。同一出口 IP 在一天内会重复出现，缓存能把文件查询压到第一次：

```java
String cacheKey = "geo:ip:" + ip;
String cached = redis.get(cacheKey);
if (cached != null) {
    record.setRegion(cached);
} else {
    String region = RegionLookup.locate(ip);
    if (region != null) {
        redis.set(cacheKey, region, Duration.ofHours(12));
        record.setRegion(region);
    }
}
```

键上带 `geo:ip:` 前缀，避免和会话、验证码抢同一个键空间。过期时间不要长到地址库更新了还在用旧结果；离线库本身几个月才会换一版，12 小时对登录日志足够。

## 打进 jar 之后，资源路径不再是文件

上面的 `resource.getPath()` 在 IDE 里通常是磁盘上的真实文件，所以本地能过。打成可执行 jar 部署后，资源在 jar 内部，URL 形态类似 `jar:file:/opt/app.jar!/geo/ip2region.db`。把这段字符串交给 `new File` 或 `DbSearcher`，会报数据文件非法，查询全部失败。

查询器这一代 API 要的是文件系统路径，不接受输入流。处理办法是第一次使用时把 classpath 里的字节抄到工作目录下的可写文件，之后发现文件已在就直接复用。

```java
static Path materialize() throws IOException {
    Path dir = Path.of(System.getProperty("user.dir"), "var", "geo");
    Path target = dir.resolve("ip2region.db");
    if (Files.isRegularFile(target) && Files.size(target) > 0) {
        return target;
    }
    Files.createDirectories(dir);
    try (InputStream in = Thread.currentThread()
            .getContextClassLoader()
            .getResourceAsStream("geo/ip2region.db")) {
        if (in == null) {
            throw new IOException("geo/ip2region.db not on classpath");
        }
        Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
    }
    return target;
}
```

`locate` 改为使用这份落地文件：

```java
public static String locate(String ip) {
    if (!Util.isIpAddress(ip)) {
        return null;
    }
    try {
        Path db = materialize();
        DbSearcher searcher = new DbSearcher(new DbConfig(), db.toString());
        DataBlock block = searcher.btreeSearch(ip);
        return tidy(block.getRegion());
    } catch (Exception ex) {
        log.warn("region lookup failed for {}", ip, ex);
        return null;
    }
}
```

重新打包部署后，进程工作目录下会出现 `var/geo/ip2region.db`，同一 IP 能再次得到地区串。注意几个容易回退的细节：

- 用上下文类加载器读资源，路径不要以 `/` 开头；`getResourceAsStream("geo/ip2region.db")` 才对。类对象的 `getResourceAsStream` 则要写成绝对资源名。两种 API 混用，会在 jar 里得到 `null` 流。
- 目标文件已存在但长度为 0（上次复制写到一半进程退出）时，应覆盖而不是直接拿去查询。上面用 `size() > 0` 做了这道判断。
- 多实例部署如果共享同一个工作目录，复制要避免互相截断。单机单进程的登录服务按上面的写法足够；多进程共目录时改成“写临时文件再原子移动”。
- 数据文件更新后，只替换 jar 不会自动更新已经抄出去的那份。发版脚本里删掉 `var/geo/ip2region.db`，或在文件名里带上数据版本。
- 不要把抄出来的 `var/` 提交进仓库，它是运行时产物。

内存算法如果改成常驻，应在启动时建一个可复用的 `DbSearcher`，而不是每次查询都把整库再装一遍。B 树模式下搜索器也尽量复用，避免每次打开文件。复用搜索器时要确认所用版本是否允许并发查询；不允许的话，用线程本地或在调用外加一把锁，不要回到“每个请求 new 一个并整库读入”的写法。

查询器持有文件句柄，进程退出或热更新数据文件之前要关掉它，否则替换 db 时会出现文件占用。可以把 `DbSearcher` 包进一个单例，在销毁回调里 `close`。另外，`tidy` 只去掉占位的 `0` 段，不要把合法的数字地名误删；如果某一段本身就是 `0` 以外的空，库里通常已经用竖线留下空位，按段拆开再过滤比全局替换更稳。日志里不要打印完整 IP 加地区之外的原始报文，避免把内部网络地址和运营商信息写进级别过低的访问日志。查询失败时返回空值，让调用方记成“未知地区”，不要把异常栈直接写进面向用户的页面。空 IP 和回环地址也应在进库之前就拦住。
