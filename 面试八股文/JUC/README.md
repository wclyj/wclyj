# JUC 并发与 JVM 笔记目录

## 阅读说明

这一组笔记按面试里常见的追问顺序整理：先把内存可见性讲清楚，再谈原子类和锁，然后是线程协作、队列、线程池，最后补 JVM 与线上排查。每一篇都可以单独复习，代码示例都改写成可独立运行的小程序。

离线阅读时直接打开对应目录下的 Markdown 即可，笔记之间用相对路径互相跳转。

## JUC

- [可见性：volatile 与 JMM](./1_谈谈Volatile/1_Volatile和JMM内存模型的可见性)
- [volatile 为什么挡不住 number++](./1_谈谈Volatile/2_Volatile不保证原子性)
- [volatile 如何挡住指令重排](./1_谈谈Volatile/3_Volatile禁止指令重排)
- [双重检查单例里为什么还要 volatile](./1_谈谈Volatile/4_Volatile的应用)
- [CAS 在原子类里怎么落地](./2_谈谈CAS/5_CAS底层原理)
- [ABA：值没变不代表没被人改过](./3_谈谈原子类的ABA问题/6_原子类AtomicInteger的ABA问题)
- [ArrayList 并发写会坏在哪里](./4_ArrayList为什么线程不安全/ArrayList线程不安全的举例)
- [方法参数到底传的是什么](./5_TransferValue是什么)
- [读写锁：读可以并行，写必须独占](./6_Java的锁/Java锁之读写锁)
- [公平锁与非公平锁](./6_Java的锁/Java锁之公平锁和非公平锁)
- [可重入锁：同一线程可以再次进入](./6_Java的锁/Java锁之可重入锁和递归锁)
- [自旋锁：用循环换上下文切换](./6_Java的锁/Java锁之自旋锁)
- [synchronized 不禁重排，为何仍有序](./6_Java的锁/Synchronized无法禁止指令重排却能保证有序性)
- [CountDownLatch：倒数到零再放行](./7_CountDownLatch_CyclicBarrier_Semaphore使用/CountDownLatch)
- [CyclicBarrier：凑齐人数再一起走](./7_CountDownLatch_CyclicBarrier_Semaphore使用/CyclicBarrier)
- [Semaphore：同时只放行固定名额](./7_CountDownLatch_CyclicBarrier_Semaphore使用/Semaphore)
- [阻塞队列四种进队出队语义](./8_阻塞队列)
- [synchronized 与 Lock 怎么选](./9_Synchronized和Lock的区别与好处)
- [线程池参数与拒绝策略](./10_线程池)
- [死锁怎么写出来，又怎么定位](./11_死锁编码及快速定位)
- [JVM 运行时区域一张图](./12_JVM/JVM体系结构)
- [GC Roots 与可达性分析](./12_JVM/JVM面试题汇总/1_什么是GCRoots能做什么)
- [常用 JVM 参数怎么看、怎么改](./12_JVM/JVM面试题汇总/2_JVM参数调优)
- [强、软、弱、虚四种引用](./12_JVM/JVM面试题汇总/3_Java中的强引用_软引用_弱引用_虚引用分别是什么)
- [几种典型的内存溢出](./12_JVM/JVM面试题汇总/4_Java内存溢出OOM)
- [垃圾收集器怎么搭配](./12_JVM/JVM面试题汇总/5_垃圾回收器)
- [Linux 上定位慢请求](./13_Linux相关命令)
- [在 GitHub 上缩小搜索范围](./14_Github学习)
- [乐观锁与悲观锁](./15_乐观锁和悲观锁)
