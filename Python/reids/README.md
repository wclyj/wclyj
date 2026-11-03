# 用 Python 访问 Redis

Redis 是进程外的键值存储，适合放缓存、计数器和短期会话。Python 里常用 `redis` 这个客户端库，连接参数（地址、端口、库号、密码）放在配置里，不要写死在业务代码中。

下面的例子不依赖某个固定键名，演示字符串过期、哈希字段，以及用流水线把多次写合并成一次网络往返。

```python
import redis

def demo(url):
    client = redis.Redis.from_url(url, decode_responses=True)
    # 会话令牌 30 秒后失效；重复执行会覆盖同一键
    client.set("session:u17", "token-ax9", ex=30)
    token = client.get("session:u17")

    # 一个用户的资料拆成多个字段，避免整包覆盖
    client.hset("user:17", mapping={"city": "hangzhou", "level": "3"})
    level = client.hget("user:17", "level")

    pipe = client.pipeline(transaction=False)
    pipe.incr("metric:login")
    pipe.expire("metric:login", 86400)
    pipe.execute()
    return token, level
```

`decode_responses=True` 让读到的是字符串而不是字节。`ex` 的单位是秒。哈希的 `hset` 只改列出的字段。流水线减少往返，但 `transaction=False` 时不保证这两条命令原子执行；若计数和设置过期必须一起成功，再用事务或 Lua。

连接失败会抛异常，调用方应按超时重试或降级，不要假设本机 6379 一定开着。键要有业务前缀和过期时间，否则缓存会一直占用内存。
