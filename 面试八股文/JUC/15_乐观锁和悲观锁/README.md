# 乐观锁和悲观锁

## 乐观锁

### 概念

乐观锁默认冲突很少。读和改的过程先不上互斥锁，写回时再核对版本还在不在。核对失败就说明别人已经写过，本次作废，调用方重读再试。表上多一列 `revision` 充当版本。成功写入时版本加一，失败则影响 0 行。

### 乐观锁实现方式

- 读出记录时记下当前版本，例如 `revision = 3`
- 提交更新时把这个版本放进条件
- SQL 写成：新版本等于旧版本加一，并且旧版本必须仍是刚才读到的值
- 版本对不上，更新影响 0 行，业务层决定重试或提示冲突

```sql
-- 先读到 revision = 3。乙若已写成 4，甲这次影响 0 行
update sku set stock = 18, revision = revision + 1
where sku_id = 42 and revision = 3;
```

## MybatisPlus 使用乐观锁

表上增加整型版本列，插入时默认写成 1。实体上用版本注解标出这一列：

```java
// 交给拦截器在更新时自动比较并递增
@Version
private Integer revision;
```

配置类注册插件。3.4 之后的类名是 `OptimisticLockerInnerInterceptor`：

```java
@Bean
public MybatisPlusInterceptor mpInterceptor() {
    MybatisPlusInterceptor chain = new MybatisPlusInterceptor();
    chain.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return chain;
}
```

## 悲观锁

悲观锁默认冲突一定会发生。读或改之前先锁住记录，例如 `select ... for update`。别人要么等待，要么拿不到锁。并发度下降，锁持有太久还会堵住后面的事务。
