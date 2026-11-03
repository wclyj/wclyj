# 按统一前缀清掉一批缓存键

## 背景

活动页把每月榜单放进 Redis，键都长成 `campaign:rank:202601` 这种样子，只是月份不同。运营下线活动时要按前缀整批清掉，但服务端没有“删除某个前缀”这种命令。

## 做法

可以先按模式把命中的键收集出来，再交给删除接口：

```java
Set<String> hit = stringRedisTemplate.keys("campaign:rank:*");
if (hit != null && !hit.isEmpty()) {
    stringRedisTemplate.delete(hit);
}
```

## 风险

`KEYS` 在扫描完成前会挡住实例，其它命令都得排队。键很多或线上流量高时，这种清理会把延迟一下子打高。
