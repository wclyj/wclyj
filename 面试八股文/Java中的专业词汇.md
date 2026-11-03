1. ReentrantLock：同一线程可反复取得的锁
2. Reference：指向对象的引用
3. concurrent：多线程同时推进
4. CountDownLatch：倒数到零才放行
5. CyclicBarrier：线程彼此等到齐再继续
6. Semaphore：用许可数限制并发
7. ThreadPoolExecutor：线程池的实现类
8. Scheduled：延迟或按周期触发
9. Serial：单线程回收器
10. Parallel：多线程回收器
11. Tenured：年老区，同 old
12. Young generation：新对象所在的代
13. Tenured generation：多次回收后进入的代
14. Garbage Collection：收回不再使用的对象
15. Garbage First：优先收垃圾多的区
16. actuator：执行任务的部件
17. nacos：注册与配置中心
18. Channel：数据进出的通道
19. capacity：最大容量
20. annotation：附在程序上的元数据
21. retention：元数据保留到哪个阶段
22. deprecated：已过时，不建议调用
23. Reflection：运行时查看并调用
24. constructor：创建对象的构造方法
25. property：对象的一项属性
26. declared：类中声明的成员，含非公开
27. invoke：发起调用
28. signal：唤醒等待中的线程
29. yield：让出本轮时间片
30. interrupt：打上中断标记
31. Aspect：横切多个方法的切面
32. atomicity：要么全成功，要么全不做
33. consistency：前后约束仍成立
34. isolation：看不见彼此的中间态
35. durability：提交后不丢
36. transaction：一起提交或一起撤销的操作
37. dispatch：把请求分给处理者
38. actuator：暴露运行状态的端点
39. availability：故障时仍提供服务
40. consistency：读到的是最近一次写入
41. Partition Tolerance：网络切开仍可做决定
42. Balancer：把流量分到多机
43. hystrix：失败过多就断开调用
44. sentinel：限流、熔断和降级组件
45. circuitBreaker：连续失败后暂停调用
46. predicate：只回答真或假
47. AES：加解密共用一把密钥
48. RSA：公钥加密、私钥解密
49. myisam：表锁、偏读取的引擎
50. InnoDB：支持事务和行锁的引擎
51. sleuth：给一次请求串起标记
52. seata：跨服务事务协调
53. transaction：库事务
54. SnowFlake：用时间、机器和序列拼号
55. confirm：代理收下消息后的回执
56. redis replication：主节点把写同步到从节点
57. Redis Cluster：数据分片后的多节点部署
58. proxy：代真正对象处理请求的一层
