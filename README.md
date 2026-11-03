# 学习笔记

## 怎么读

笔记按主题分目录。算法题一般是题目、思路、手算例子、代码、复杂度和反例；组件和运维类笔记按概念、步骤、配置和易错点来写。例子、样例数据和解法代码都是按这个主题重写的，用来把机制走通，而不是照搬某一份现成讲义。

## 覆盖范围

目录从 Java 与并发、JVM、Spring 全家桶，到 MySQL、Redis、容器和 Kubernetes，再到 Go、前端、网络、操作系统、数据结构、LeetCode 和集成学习。下面的索引按主题把每一篇链出来。

参考目录里有一个 `面试题` 夹，里面是两份二进制 PDF，这里不收录。面试向的内容写在「面经」「中间件」和「面试补充」里。`NuxtJS` 在参考目录中只有空占位，没有笔记。

## Java

- [1_使用了并发工具类库，线程安全就高枕无忧了吗？](Java/1_使用了并发工具类库，线程安全就高枕无忧了吗？/README.md)
- [2_代码加锁，不要让“锁”成为烦心事](Java/2_代码加锁，不要让“锁”成为烦心事/README.md)
- [3_线程池：业务代码最常用也最容易犯错的组件](Java/3_线程池：业务代码最常用也最容易犯错的组件/README.md)
- [4_连接池：别让连接池帮了倒忙](Java/4_连接池：别让连接池帮了倒忙/README.md)
- [5_HTTP调用：你考虑到超时、重试、并发了吗？](Java/5_HTTP调用：你考虑到超时、重试、并发了吗？/README.md)
- [6_20%的业务代码的Spring声明式事务，可能都没处理正确](Java/6_20%的业务代码的Spring声明式事务，可能都没处理正确/README.md)
- [7_数据库索引：索引并不是万能药](Java/7_数据库索引：索引并不是万能药/README.md)
- [8_判等问题：程序里如何确定你就是你？](Java/8_判等问题：程序里如何确定你就是你？/README.md)
- [9_数值计算：注意精度、舍入和溢出问题](Java/9_数值计算：注意精度、舍入和溢出问题/README.md)
- [ArrayList扩容机制](Java/ArrayList扩容机制/README.md)
- [JVM类加载机制](Java/JVM类加载机制/README.md)
- [Java中的双亲委派机制以及如何打破](Java/Java中的双亲委派机制以及如何打破/README.md)
- [Java使用Redis删除指定前缀Key](Java/Java使用Redis删除指定前缀Key/README.md)
- [Java注解和反射](Java/Java注解和反射/README.md)
- [VisualVM安装VisualGC插件](Java/VisualVM安装VisualGC插件/README.md)
- [使用Ip2region替代淘宝IP接口](Java/使用Ip2region替代淘宝IP接口/README.md)
- [前端的一些跨域问题](Java/前端的一些跨域问题/README.md)
- [泛型的类型擦除](Java/泛型的类型擦除/README.md)
- [谈谈你对AQS的理解](Java/谈谈你对AQS的理解/README.md)
- [谈谈你对ThreadLocal的理解](Java/谈谈你对ThreadLocal的理解/README.md)
- [1_equals和等等的区别](面试八股文/基础面试题/1_equals和等等的区别/README.md)
- [2_代码块](面试八股文/基础面试题/2_代码块/README.md)
- [3_分布式锁](面试八股文/基础面试题/3_分布式锁/README.md)
- [4_MySQL的存储引擎](面试八股文/基础面试题/4_MySQL的存储引擎/README.md)
- [5_JDK动态代理和CGLIB动态代理](面试八股文/基础面试题/5_JDK动态代理和CGLIB动态代理/README.md)

## Java 8 新特性

- [1_HashMap变化](面试八股文/Java8新特性/1_HashMap变化/README.md)
- [2_Lambda表达式](面试八股文/Java8新特性/2_Lambda表达式/README.md)
- [3_方法引用和构造器](面试八股文/Java8新特性/3_方法引用和构造器/README.md)
- [4_强大的Stream](面试八股文/Java8新特性/4_强大的Stream/README.md)
- [5_并行流](面试八股文/Java8新特性/5_并行流/README.md)
- [6_Optional容器类](面试八股文/Java8新特性/6_Optional容器类/README.md)

## NIO

- [NIO是什么](面试八股文/NIO/NIO是什么/README.md)
- [NIO的使用案例](面试八股文/NIO/NIO的使用案例/README.md)

## JVM

- [10_对象实例化内存布局与访问定位](JVM/1_内存与垃圾回收篇/10_对象实例化内存布局与访问定位/README.md)
- [11_直接内存](JVM/1_内存与垃圾回收篇/11_直接内存/README.md)
- [12_执行引擎](JVM/1_内存与垃圾回收篇/12_执行引擎/README.md)
- [13_StringTable](JVM/1_内存与垃圾回收篇/13_StringTable/README.md)
- [14_垃圾回收概述](JVM/1_内存与垃圾回收篇/14_垃圾回收概述/README.md)
- [15_垃圾回收相关算法](JVM/1_内存与垃圾回收篇/15_垃圾回收相关算法/README.md)
- [16_垃圾回收相关概念](JVM/1_内存与垃圾回收篇/16_垃圾回收相关概念/README.md)
- [17_垃圾回收器](JVM/1_内存与垃圾回收篇/17_垃圾回收器/README.md)
- [1_JVM与Java体系结构](JVM/1_内存与垃圾回收篇/1_JVM与Java体系结构/README.md)
- [2_类加载子系统](JVM/1_内存与垃圾回收篇/2_类加载子系统/README.md)
- [3_运行时数据区概述及线程](JVM/1_内存与垃圾回收篇/3_运行时数据区概述及线程/README.md)
- [4_程序计数器](JVM/1_内存与垃圾回收篇/4_程序计数器/README.md)
- [5_虚拟机栈](JVM/1_内存与垃圾回收篇/5_虚拟机栈/README.md)
- [6_本地方法接口](JVM/1_内存与垃圾回收篇/6_本地方法接口/README.md)
- [7_本地方法栈](JVM/1_内存与垃圾回收篇/7_本地方法栈/README.md)
- [8_堆](JVM/1_内存与垃圾回收篇/8_堆/README.md)
- [9_方法区](JVM/1_内存与垃圾回收篇/9_方法区/README.md)
- [JVM](JVM/README.md)

## JUC

- [10_线程池](面试八股文/JUC/10_线程池/README.md)
- [11_死锁编码及快速定位](面试八股文/JUC/11_死锁编码及快速定位/README.md)
- [JVM体系结构](面试八股文/JUC/12_JVM/JVM体系结构/README.md)
- [1_什么是GCRoots能做什么](面试八股文/JUC/12_JVM/JVM面试题汇总/1_什么是GCRoots能做什么/README.md)
- [2_JVM参数调优](面试八股文/JUC/12_JVM/JVM面试题汇总/2_JVM参数调优/README.md)
- [3_Java中的强引用_软引用_弱引用_虚引用分别是什么](面试八股文/JUC/12_JVM/JVM面试题汇总/3_Java中的强引用_软引用_弱引用_虚引用分别是什么/README.md)
- [4_Java内存溢出OOM](面试八股文/JUC/12_JVM/JVM面试题汇总/4_Java内存溢出OOM/README.md)
- [5_垃圾回收器](面试八股文/JUC/12_JVM/JVM面试题汇总/5_垃圾回收器/README.md)
- [13_Linux相关命令](面试八股文/JUC/13_Linux相关命令/README.md)
- [14_Github学习](面试八股文/JUC/14_Github学习/README.md)
- [15_乐观锁和悲观锁](面试八股文/JUC/15_乐观锁和悲观锁/README.md)
- [1_Volatile和JMM内存模型的可见性](面试八股文/JUC/1_谈谈Volatile/1_Volatile和JMM内存模型的可见性/README.md)
- [2_Volatile不保证原子性](面试八股文/JUC/1_谈谈Volatile/2_Volatile不保证原子性/README.md)
- [3_Volatile禁止指令重排](面试八股文/JUC/1_谈谈Volatile/3_Volatile禁止指令重排/README.md)
- [4_Volatile的应用](面试八股文/JUC/1_谈谈Volatile/4_Volatile的应用/README.md)
- [5_CAS底层原理](面试八股文/JUC/2_谈谈CAS/5_CAS底层原理/README.md)
- [6_原子类AtomicInteger的ABA问题](面试八股文/JUC/3_谈谈原子类的ABA问题/6_原子类AtomicInteger的ABA问题/README.md)
- [ArrayList线程不安全的举例](面试八股文/JUC/4_ArrayList为什么线程不安全/ArrayList线程不安全的举例/README.md)
- [5_TransferValue是什么](面试八股文/JUC/5_TransferValue是什么/README.md)
- [Java锁之公平锁和非公平锁](面试八股文/JUC/6_Java的锁/Java锁之公平锁和非公平锁/README.md)
- [Java锁之可重入锁和递归锁](面试八股文/JUC/6_Java的锁/Java锁之可重入锁和递归锁/README.md)
- [Java锁之自旋锁](面试八股文/JUC/6_Java的锁/Java锁之自旋锁/README.md)
- [Java锁之读写锁](面试八股文/JUC/6_Java的锁/Java锁之读写锁/README.md)
- [Synchronized无法禁止指令重排却能保证有序性](面试八股文/JUC/6_Java的锁/Synchronized无法禁止指令重排却能保证有序性/README.md)
- [CountDownLatch](面试八股文/JUC/7_CountDownLatch_CyclicBarrier_Semaphore使用/CountDownLatch/README.md)
- [CyclicBarrier](面试八股文/JUC/7_CountDownLatch_CyclicBarrier_Semaphore使用/CyclicBarrier/README.md)
- [Semaphore](面试八股文/JUC/7_CountDownLatch_CyclicBarrier_Semaphore使用/Semaphore/README.md)
- [8_阻塞队列](面试八股文/JUC/8_阻塞队列/README.md)
- [9_Synchronized和Lock的区别与好处](面试八股文/JUC/9_Synchronized和Lock的区别与好处/README.md)
- [JUC](面试八股文/JUC/README.md)

## Docker

- [DockerCompose入门学习](Docker/DockerCompose入门学习/README.md)
- [DockerDesktop入门简介](Docker/DockerDesktop入门简介/README.md)
- [Docker图形化工具Portainer介绍与安装](Docker/Docker图形化工具Portainer介绍与安装/README.md)
- [Docker容器相关命令](Docker/Docker容器相关命令/README.md)
- [Docker操作系统之Alpine](Docker/Docker操作系统之Alpine/README.md)
- [Docker最新的镜像库](Docker/Docker最新的镜像库/README.md)
- [如何将镜像推送到阿里云容器镜像服务](Docker/如何将镜像推送到阿里云容器镜像服务/README.md)
- [对象存储MinIO入门简介](Docker/对象存储MinIO入门简介/README.md)

## K8S

- [10_Kubernetes核心技术Service](K8S/10_Kubernetes核心技术Service/README.md)
- [11_Kubernetes控制器Controller详解](K8S/11_Kubernetes控制器Controller详解/README.md)
- [12_Kubernetes配置管理](K8S/12_Kubernetes配置管理/README.md)
- [13_Kubernetes集群安全机制](K8S/13_Kubernetes集群安全机制/README.md)
- [14_Kubernetes核心技术Ingress](K8S/14_Kubernetes核心技术Ingress/README.md)
- [15_Kubernetes核心技术Helm](K8S/15_Kubernetes核心技术Helm/README.md)
- [16_Kubernetes持久化存储](K8S/16_Kubernetes持久化存储/README.md)
- [17_Kubernetes集群资源监控](K8S/17_Kubernetes集群资源监控/README.md)
- [18_Kubernetes搭建高可用集群](K8S/18_Kubernetes搭建高可用集群/README.md)
- [19_Kubernetes容器交付介绍](K8S/19_Kubernetes容器交付介绍/README.md)
- [1_Kubernetes简介](K8S/1_Kubernetes简介/README.md)
- [2_搭建K8S集群前置知识](K8S/2_搭建K8S集群前置知识/README.md)
- [30_使用kubeadm-ha脚本一键安装K8S](K8S/30_使用kubeadm-ha脚本一键安装K8S/README.md)
- [31_Kubernetes可视化界面kubesphere](K8S/31_Kubernetes可视化界面kubesphere/README.md)
- [32_Kubernetes配置默认存储类](K8S/32_Kubernetes配置默认存储类/README.md)
- [3_使用kubeadm方式搭建K8S集群](K8S/3_使用kubeadm方式搭建K8S集群/README.md)
- [4_使用二进制方式搭建K8S集群](K8S/4_使用二进制方式搭建K8S集群/README.md)
- [50_使用Rancher搭建Kubernetes集群](K8S/50_使用Rancher搭建Kubernetes集群/README.md)
- [52_Kubernetes中的CRI](K8S/52_Kubernetes中的CRI/README.md)
- [53_Helm入门](K8S/53_Helm入门/README.md)
- [5_Kubeadm和二进制方式对比](K8S/5_Kubeadm和二进制方式对比/README.md)
- [6_Kubernetes集群管理工具kubectl](K8S/6_Kubernetes集群管理工具kubectl/README.md)
- [7_Kubernetes集群YAML文件详解](K8S/7_Kubernetes集群YAML文件详解/README.md)
- [8_Kubernetes核心技术Pod](K8S/8_Kubernetes核心技术Pod/README.md)
- [9_Kubernetes核心技术Controller](K8S/9_Kubernetes核心技术Controller/README.md)
- [1_云平台核心](K8S/Kubesphere/1_云平台核心/README.md)
- [2_Docker基础](K8S/Kubesphere/2_Docker基础/README.md)
- [3_Kubernetes入门](K8S/Kubesphere/3_Kubernetes入门/README.md)
- [Kubesphere](K8S/Kubesphere/README.md)
- [K8S](K8S/README.md)

## 中间件

- [10_设计一个高并发系统](面试八股文/面试扫盲学习/10_设计一个高并发系统/README.md)
- [11_数据库分库分表的面试连环炮](面试八股文/面试扫盲学习/11_数据库分库分表的面试连环炮/README.md)
- [12_MySQL读写复制及主从同步时延](面试八股文/面试扫盲学习/12_MySQL读写复制及主从同步时延/README.md)
- [13_如何设计高可用系统架构](面试八股文/面试扫盲学习/13_如何设计高可用系统架构/README.md)
- [1_消息队列的面试连环炮](面试八股文/面试扫盲学习/1_消息队列的面试连环炮/README.md)
- [2_分布式搜索引擎的面试连环炮](面试八股文/面试扫盲学习/2_分布式搜索引擎的面试连环炮/README.md)
- [3_分布式缓存](面试八股文/面试扫盲学习/3_分布式缓存/README.md)
- [4_Redis的面试连环炮](面试八股文/面试扫盲学习/4_Redis的面试连环炮/README.md)
- [5_Redis的面试连环炮2](面试八股文/面试扫盲学习/5_Redis的面试连环炮2/README.md)
- [6_分布式系统的面试连环炮](面试八股文/面试扫盲学习/6_分布式系统的面试连环炮/README.md)
- [7_分布式系统幂等性与顺序性及分布式锁](面试八股文/面试扫盲学习/7_分布式系统幂等性与顺序性及分布式锁/README.md)
- [8_分布式Session解决方案](面试八股文/面试扫盲学习/8_分布式Session解决方案/README.md)
- [9_Spring中的事务](面试八股文/面试扫盲学习/9_Spring中的事务/README.md)
- [9_分布式事务解决方案](面试八股文/面试扫盲学习/9_分布式事务解决方案/README.md)
- [面试扫盲学习](面试八股文/面试扫盲学习/README.md)

## Spring Cloud

- [API网关](SpringCloud/API网关/README.md)
- [Feign配置全局Sentinel流控异常和降级异常](SpringCloud/Feign配置全局Sentinel流控异常和降级异常/README.md)
- [Nacos关闭心跳日志](SpringCloud/Nacos关闭心跳日志/README.md)
- [10_消息总线Bus](SpringCloud/SpringCloud2020/10_消息总线Bus/README.md)
- [11_消息驱动SpringCloudStream](SpringCloud/SpringCloud2020/11_消息驱动SpringCloudStream/README.md)
- [12_SpringCloudSleuth分布式请求链路跟踪](SpringCloud/SpringCloud2020/12_SpringCloudSleuth分布式请求链路跟踪/README.md)
- [13_Nacos是什么](SpringCloud/SpringCloud2020/13_Nacos是什么/README.md)
- [14_SpringCloudAlibabaSentinel实现熔断和限流](SpringCloud/SpringCloud2020/14_SpringCloudAlibabaSentinel实现熔断和限流/README.md)
- [15_SpringCloudAlibabaSeata处理分布式事务](SpringCloud/SpringCloud2020/15_SpringCloudAlibabaSeata处理分布式事务/README.md)
- [1_SpringCloud是什么](SpringCloud/SpringCloud2020/1_SpringCloud是什么/README.md)
- [2_搭建一个SpringCloud项目](SpringCloud/SpringCloud2020/2_搭建一个SpringCloud项目/README.md)
- [3_搭建Eureka集群](SpringCloud/SpringCloud2020/3_搭建Eureka集群/README.md)
- [4_Eureka停更后的替换](SpringCloud/SpringCloud2020/4_Eureka停更后的替换/README.md)
- [5_Ribbon负载均衡](SpringCloud/SpringCloud2020/5_Ribbon负载均衡/README.md)
- [6_OpenFeign实现服务调用](SpringCloud/SpringCloud2020/6_OpenFeign实现服务调用/README.md)
- [7_Hystrix中的服务降级和熔断](SpringCloud/SpringCloud2020/7_Hystrix中的服务降级和熔断/README.md)
- [8_服务网关Gateway](SpringCloud/SpringCloud2020/8_服务网关Gateway/README.md)
- [9_分布式配置中心SpringCloudConfig](SpringCloud/SpringCloud2020/9_分布式配置中心SpringCloudConfig/README.md)
- [SpringCloud2020](SpringCloud/SpringCloud2020/README.md)
- [使用Zipkin搭建蘑菇博客链路追踪](SpringCloud/使用Zipkin搭建蘑菇博客链路追踪/README.md)
- [微服务之吐槽一下Nacos日志的疯狂输出](SpringCloud/微服务之吐槽一下Nacos日志的疯狂输出/README.md)
- [新建文件夹](SpringCloud/新建文件夹/README.md)
- [配置Sentinel规则持久化到Nacos中](SpringCloud/配置Sentinel规则持久化到Nacos中/README.md)

## Spring Security

- [1_初识SpringSecurity](SpringSecurity/1_初识SpringSecurity/README.md)
- [2_SpringSecurity在MVC项目中的使用](SpringSecurity/2_SpringSecurity在MVC项目中的使用/README.md)
- [3_SpringSecurity在单机环境下的使用](SpringSecurity/3_SpringSecurity在单机环境下的使用/README.md)
- [4_SpringSecurity在分布式环境下的使用](SpringSecurity/4_SpringSecurity在分布式环境下的使用/README.md)
- [5_OAuth2.0介绍](SpringSecurity/5_OAuth2.0介绍/README.md)
- [SpringSecurity](SpringSecurity/README.md)

## Elastic Stack

- [1_ElasticSearch介绍与安装](ElasticStack/1_ElasticSearch介绍与安装/README.md)
- [2_Beats入门简介](ElasticStack/2_Beats入门简介/README.md)
- [3_Kibana安装与介绍](ElasticStack/3_Kibana安装与介绍/README.md)
- [4_Logstash入门简介](ElasticStack/4_Logstash入门简介/README.md)
- [5_ElasticStack综合案例](ElasticStack/5_ElasticStack综合案例/README.md)
- [6_使用ELK搭建蘑菇博客日志收集](ElasticStack/6_使用ELK搭建蘑菇博客日志收集/README.md)
- [ElasticStack](ElasticStack/README.md)

## 算法与数据结构

- [10_栈的压入弹出序列](数据结构/10_栈的压入弹出序列/README.md)
- [11_从尾到头打印链表](数据结构/11_从尾到头打印链表/README.md)
- [12_链表中倒数第K个节点](数据结构/12_链表中倒数第K个节点/README.md)
- [13_反转链表](数据结构/13_反转链表/README.md)
- [14_合并两个排序的链表](数据结构/14_合并两个排序的链表/README.md)
- [15_复杂链表的复制](数据结构/15_复杂链表的复制/README.md)
- [16_两个链表的公共结点](数据结构/16_两个链表的公共结点/README.md)
- [17_孩子们的游戏(圆圈中最后剩下的数)](数据结构/17_孩子们的游戏%28圆圈中最后剩下的数%29/README.md)
- [18_链表中环的入口结点](数据结构/18_链表中环的入口结点/README.md)
- [19_二进制中1的个数](数据结构/19_二进制中1的个数/README.md)
- [1_斐波那契数列](数据结构/1_斐波那契数列/README.md)
- [20_不用加减乘除做加法](数据结构/20_不用加减乘除做加法/README.md)
- [21_数组中出现次数超过一半的数字](数据结构/21_数组中出现次数超过一半的数字/README.md)
- [22_整数中1出现的次数](数据结构/22_整数中1出现的次数/README.md)
- [23_数组中只出现一次的数字](数据结构/23_数组中只出现一次的数字/README.md)
- [24_树的遍历](数据结构/24_树的遍历/README.md)
- [25_重建二叉树](数据结构/25_重建二叉树/README.md)
- [26_树的子结构](数据结构/26_树的子结构/README.md)
- [27_二叉树的镜像](数据结构/27_二叉树的镜像/README.md)
- [28_从上往下打印二叉树](数据结构/28_从上往下打印二叉树/README.md)
- [29_二叉搜索树的后序遍历序列](数据结构/29_二叉搜索树的后序遍历序列/README.md)
- [2_青蛙跳台阶](数据结构/2_青蛙跳台阶/README.md)
- [30_二叉树中和为某一值的路径](数据结构/30_二叉树中和为某一值的路径/README.md)
- [31_二叉搜索树与双向链表](数据结构/31_二叉搜索树与双向链表/README.md)
- [32_最小的K个数](数据结构/32_最小的K个数/README.md)
- [33_数据流中的中位数](数据结构/33_数据流中的中位数/README.md)
- [34_二叉树的下一个节点](数据结构/34_二叉树的下一个节点/README.md)
- [35_对称的二叉树](数据结构/35_对称的二叉树/README.md)
- [36_按之字形顺序打印二叉树](数据结构/36_按之字形顺序打印二叉树/README.md)
- [37_把二叉树打印成多行](数据结构/37_把二叉树打印成多行/README.md)
- [38_二叉搜索树的第K个节点](数据结构/38_二叉搜索树的第K个节点/README.md)
- [39_序列化二叉树](数据结构/39_序列化二叉树/README.md)
- [3_找出丑数](数据结构/3_找出丑数/README.md)
- [40_连续子数组的最大和](数据结构/40_连续子数组的最大和/README.md)
- [41_矩形覆盖](数据结构/41_矩形覆盖/README.md)
- [42_排序算法-冒泡插入选择](数据结构/42_排序算法-冒泡插入选择/README.md)
- [43_希尔排序](数据结构/43_希尔排序/README.md)
- [44_归并排序](数据结构/44_归并排序/README.md)
- [45_快速排序](数据结构/45_快速排序/README.md)
- [4_二维数组中的查找](数据结构/4_二维数组中的查找/README.md)
- [5_替换空格](数据结构/5_替换空格/README.md)
- [6_两个栈实现一个队列](数据结构/6_两个栈实现一个队列/README.md)
- [7_旋转数组的最小数字](数据结构/7_旋转数组的最小数字/README.md)
- [8_调整数组顺序使奇数位于偶数前面](数据结构/8_调整数组顺序使奇数位于偶数前面/README.md)
- [9_包含min函数的栈](数据结构/9_包含min函数的栈/README.md)
- [KMP算法](数据结构/KMP算法/README.md)
- [剑指offer1-24题 .md](数据结构/剑指offer/剑指offer1-24题%20.md)
- [剑指offer25-50题.md](数据结构/剑指offer/剑指offer25-50题.md)
- [补码.md](数据结构/剑指offer/补码.md)
- [READMNE.md](数据结构/动态规划算法/READMNE.md)
- [常见的动态规划算法](数据结构/动态规划算法/常见的动态规划算法/README.md)
- [背包问题](数据结构/动态规划算法/背包问题/README.md)
- [贪心算法](数据结构/贪心算法/README.md)

## LeetCode

- [102_二叉树的层次遍历](LeetCode/102_二叉树的层次遍历/README.md)
- [112_路径总和](LeetCode/112_路径总和/README.md)
- [1252_奇数值单元格的数目](LeetCode/1252_奇数值单元格的数目/README.md)
- [1266_访问所有点的最小时间](LeetCode/1266_访问所有点的最小时间/README.md)
- [1295_统计位数为偶数的数字](LeetCode/1295_统计位数为偶数的数字/README.md)
- [1299_将每个元素替换为右侧最大元素](LeetCode/1299_将每个元素替换为右侧最大元素/README.md)
- [1313_解压编码列表](LeetCode/1313_解压编码列表/README.md)
- [1351_统计有序矩阵中的负数](LeetCode/1351_统计有序矩阵中的负数/README.md)
- [1365_有多少小于当前数字的数字](LeetCode/1365_有多少小于当前数字的数字/README.md)
- [1389_按既定顺序创建目标数组](LeetCode/1389_按既定顺序创建目标数组/README.md)
- [1431_拥有最多糖果的孩子](LeetCode/1431_拥有最多糖果的孩子/README.md)
- [1450_在既定时间做作业的学生人数](LeetCode/1450_在既定时间做作业的学生人数/README.md)
- [1460_通过翻转子数组使两个数组相等](LeetCode/1460_通过翻转子数组使两个数组相等/README.md)
- [1464_数组中两元素的最大乘积](LeetCode/1464_数组中两元素的最大乘积/README.md)
- [146_LRU缓存机制](LeetCode/146_LRU缓存机制/README.md)
- [1470_重新排列数组](LeetCode/1470_重新排列数组/README.md)
- [1480_一维数组的动态和](LeetCode/1480_一维数组的动态和/README.md)
- [1486_数组异或操作](LeetCode/1486_数组异或操作/README.md)
- [189_旋转数组](LeetCode/189_旋转数组/README.md)
- [199_二叉树的右视图](LeetCode/199_二叉树的右视图/README.md)
- [215_数组中的第K个最大元素](LeetCode/215_数组中的第K个最大元素/README.md)
- [415_字符串相加](LeetCode/415_字符串相加/README.md)

## Spring Boot

- [Bean的生命周期](SpringBoot/Bean的生命周期/README.md)
- [Eureka管理页面配置接口返回git信息](SpringBoot/Eureka管理页面配置接口返回git信息/README.md)
- [Java如何通过IP地址获取地区](SpringBoot/Java如何通过IP地址获取地区/README.md)
- [SpringBoot中使用注解的方式创建队列和交换机](SpringBoot/SpringBoot中使用注解的方式创建队列和交换机/README.md)
- [SpringBoot拦截器无法注入Bean](SpringBoot/SpringBoot拦截器无法注入Bean/README.md)
- [SpringBoot解决时区问题](SpringBoot/SpringBoot解决时区问题/README.md)
- [SpringBoot项目中使用字符串占位符](SpringBoot/SpringBoot项目中使用字符串占位符/README.md)
- [SpringSecurity造成无法使用iframe的内嵌页面的解决方法](SpringBoot/SpringSecurity造成无法使用iframe的内嵌页面的解决方法/README.md)
- [Swagger-ui.html页面出现404](SpringBoot/Swagger-ui.html页面出现404/README.md)
- [使用DevTool实现SpringBoot项目热部署](SpringBoot/使用DevTool实现SpringBoot项目热部署/README.md)
- [使用自定义日志接口收集用户访问日志](SpringBoot/使用自定义日志接口收集用户访问日志/README.md)
- [将Swagger升级3.0](SpringBoot/将Swagger升级3.0/README.md)
- [解决升级SpringBoot2.X后无法向eureka注册服务的问题](SpringBoot/解决升级SpringBoot2.X后无法向eureka注册服务的问题/README.md)

## Vue

- [Axios中拦截器的使用](Vue/Axios中拦截器的使用/README.md)
- [ElementUI中Upload如何批量上传](Vue/ElementUI中Upload如何批量上传/README.md)
- [SPA的SEO方案对比以及实践](Vue/SPA的SEO方案对比以及实践/README.md)
- [Vue2创建项目步骤](Vue/Vue2创建项目步骤/README.md)
- [Vue3创建项目步骤](Vue/Vue3创建项目步骤/README.md)
- [Vuex学习指南-实现一个计数器](Vue/VueX/Vuex学习指南-实现一个计数器/README.md)
- [my-project](Vue/VueX/Vuex学习指南-实现一个计数器/my-project/README.md)
- [Vue中Html和Markdown互相转换](Vue/Vue中Html和Markdown互相转换/README.md)
- [Vue中input框自动聚焦](Vue/Vue中input框自动聚焦/README.md)
- [Vue中使用Vue-cropper进行图片裁剪](Vue/Vue中使用Vue-cropper进行图片裁剪/README.md)
- [myproject](Vue/Vue中使用Vue-cropper进行图片裁剪/myproject/README.md)
- [Vue中对数组变化监听](Vue/Vue中对数组变化监听/README.md)
- [Vue中防止XSS脚本攻击](Vue/Vue中防止XSS脚本攻击/README.md)
- [Vue使用Echarts制作一个文章贡献度表](Vue/Vue使用Echarts制作一个文章贡献度表/README.md)
- [Vue使用vue-count-to插件对数字显示美化](Vue/Vue使用vue-count-to插件对数字显示美化/README.md)
- [Vue内容页面SEO优化方案](Vue/Vue内容页面SEO优化方案/README.md)
- [Vue制作一个评论模块](Vue/Vue制作一个评论模块/README.md)
- [Vue动态计算Table表格的高度](Vue/Vue动态计算Table表格的高度/README.md)
- [Vue如何给图片宽高设置固定比例](Vue/Vue如何给图片宽高设置固定比例/README.md)
- [Vue对Element中的e-tag添加@click事件无效](Vue/Vue对Element中的e-tag添加@click事件无效/README.md)
- [Vue记一次诡异的错误Uncaught_TypeError_Cannot read property_disabled_of_null](Vue/Vue记一次诡异的错误Uncaught_TypeError_Cannot%20read%20property_disabled_of_null/README.md)
- [Vue项目中引入markdown编辑器vditor](Vue/Vue项目中引入markdown编辑器vditor/README.md)
- [Vue项目使用阿里巴巴矢量图标库](Vue/Vue项目使用阿里巴巴矢量图标库/README.md)
- [Vue项目使用预渲染解决SEO优化问题](Vue/Vue项目使用预渲染解决SEO优化问题/README.md)
- [Vue项目如何关闭Eslint校验](Vue/Vue项目如何关闭Eslint校验/README.md)
- [Vue项目引入CDN加速](Vue/Vue项目引入CDN加速/README.md)
- [Vue项目引入侧边导航栏](Vue/Vue项目引入侧边导航栏/README.md)
- [Vue项目打包后动态配置解决方案](Vue/Vue项目打包后动态配置解决方案/README.md)
- [Vue项目的SEO解决方案](Vue/Vue项目的SEO解决方案/README.md)
- [el-select因为绑定的值为整数而无法默认选择](Vue/el-select因为绑定的值为整数而无法默认选择/README.md)
- [vue打包后的dist文件如何启动测试](Vue/vue打包后的dist文件如何启动测试/README.md)
- [使用Vuex进行两个页面逻辑交互](Vue/使用Vuex进行两个页面逻辑交互/README.md)
- [解决Vue项目打包后js文件过大的问题](Vue/解决Vue项目打包后js文件过大的问题/README.md)

## Linux

- [Asciinema-一种基于文本的终端录制方法](Linux/Asciinema-一种基于文本的终端录制方法/README.md)
- [CentOS下如何安装Nginx](Linux/CentOS下如何安装Nginx/README.md)
- [CentOS下安装Nacos](Linux/CentOS下安装Nacos/README.md)
- [CentOS下安装Sentinel](Linux/CentOS下安装Sentinel/README.md)
- [CentOS中将文本中的ip替换成服务器外网ip](Linux/CentOS中将文本中的ip替换成服务器外网ip/README.md)
- [Linux下查看文件和文件夹占用空间大小](Linux/Linux下查看文件和文件夹占用空间大小/README.md)
- [Linux下通过nginx配置https](Linux/Linux下通过nginx配置https/README.md)
- [VMWare中CentOS如何配置固定IP](Linux/VMWare中CentOS如何配置固定IP/README.md)
- [VmWare给CentOS扩容](Linux/VmWare给CentOS扩容/README.md)
- [记一次因代码出错不断输出日志占满Docker容器硬盘的排查经历](Linux/记一次因代码出错不断输出日志占满Docker容器硬盘的排查经历/README.md)

## Redis

- [IO多路复用底层原理](Redis/IO多路复用底层原理/README.md)
- [Redis中的数据结构](Redis/Redis中的数据结构/README.md)
- [Redis中的跳跃表](Redis/Redis中的跳跃表/README.md)
- [Redis实现分布式锁](Redis/Redis实现分布式锁/README.md)
- [Redis缓存穿透-布隆过滤器](Redis/Redis缓存穿透-布隆过滤器/README.md)
- [大白话谈IO模型](Redis/大白话谈IO模型/README.md)

## JavaScript

- [Js设置二级域名和顶级域名下共享Cookie](JavaScript/Js设置二级域名和顶级域名下共享Cookie/README.md)
- [如何通过Js将时间转换为刚刚_几分钟前_几小时前](JavaScript/如何通过Js将时间转换为刚刚_几分钟前_几小时前/README.md)

## 数据库

- [MyBatis常见面试题](数据库/MyBatis常见面试题/README.md)
- [MyBatis的缓存机制](数据库/MyBatis的缓存机制/README.md)
- [MySQL索引](数据库/MySQL索引/README.md)

## MySQL

- [00_开篇词 这一次，让我们一起来搞懂MySQL](MySQL/MySQL45讲/00_开篇词%20这一次，让我们一起来搞懂MySQL/README.md)
- [01_基础架构：一条SQL查询语句是如何执行的？](MySQL/MySQL45讲/01_基础架构：一条SQL查询语句是如何执行的？/README.md)
- [02_日志系统：一条SQL更新语句是如何执行的？](MySQL/MySQL45讲/02_日志系统：一条SQL更新语句是如何执行的？/README.md)
- [03_事务隔离：为什么你改了我还看不见？](MySQL/MySQL45讲/03_事务隔离：为什么你改了我还看不见？/README.md)
- [04_深入浅出索引（上）](MySQL/MySQL45讲/04_深入浅出索引（上）/README.md)
- [05_深入浅出索引（下）](MySQL/MySQL45讲/05_深入浅出索引（下）/README.md)
- [06_全局锁和表锁 ：给表加个字段怎么有这么多阻碍？](MySQL/MySQL45讲/06_全局锁和表锁%20：给表加个字段怎么有这么多阻碍？/README.md)
- [07_行锁功过：怎么减少行锁对性能的影响？](MySQL/MySQL45讲/07_行锁功过：怎么减少行锁对性能的影响？/README.md)
- [08_事务到底是隔离的还是不隔离的](MySQL/MySQL45讲/08_事务到底是隔离的还是不隔离的/README.md)
- [09_普通索引和唯一索引，应该怎么选择？](MySQL/MySQL45讲/09_普通索引和唯一索引，应该怎么选择？/README.md)
- [100_Mysql explain详解](MySQL/MySQL45讲/100_Mysql%20explain详解/README.md)
- [10_MySQL为什么有时候会选错索引](MySQL/MySQL45讲/10_MySQL为什么有时候会选错索引/README.md)
- [11_怎么给字符串字段加索引？](MySQL/MySQL45讲/11_怎么给字符串字段加索引？/README.md)
- [12_为什么我的MySQL会“抖”一下？](MySQL/MySQL45讲/12_为什么我的MySQL会“抖”一下？/README.md)
- [14_count()这么慢，我该怎么办？](MySQL/MySQL45讲/14_count%28%29这么慢，我该怎么办？/README.md)
- [15_日志和索引相关问题](MySQL/MySQL45讲/15_日志和索引相关问题/README.md)
- [16_order by是怎么工作的？](MySQL/MySQL45讲/16_order%20by是怎么工作的？/README.md)
- [17_如何正确地显示随机消息？](MySQL/MySQL45讲/17_如何正确地显示随机消息？/README.md)

## 操作系统

- [1_进程和线程通信](操作系统/1_进程和线程通信/README.md)
- [2_死锁的产生与解决](操作系统/2_死锁的产生与解决/README.md)
- [3_操作系统八股文](操作系统/3_操作系统八股文/README.md)

## 计算机网络

- [1_三次握手和四次挥手](计算机网络/1_三次握手和四次挥手/README.md)
- [2_https和http](计算机网络/2_https和http/README.md)
- [3_TCP中的拥塞控制和流量控制](计算机网络/3_TCP中的拥塞控制和流量控制/README.md)
- [4_物理层](计算机网络/4_物理层/README.md)
- [5_数据链路层](计算机网络/5_数据链路层/README.md)
- [http中的状态码](计算机网络/http中的状态码/README.md)

## 面经

- [1_京东面经](面试八股文/面经汇总/1_京东面经/README.md)
- [2_字节跳动面试总结](面试八股文/面经汇总/2_字节跳动面试总结/README.md)
- [3_京东零售提前批Java一面](面试八股文/面经汇总/3_京东零售提前批Java一面/README.md)
- [4_京东零售提前批Java二面](面试八股文/面经汇总/4_京东零售提前批Java二面/README.md)
- [5_滴滴出行提前批Java123面](面试八股文/面经汇总/5_滴滴出行提前批Java123面/README.md)

## Golang 基础

- [0_Go语言的安装](Golang/Golang基础/0_Go语言的安装/README.md)
- [10_Go中的日期函数](Golang/Golang基础/10_Go中的日期函数/README.md)
- [11_Go中的指针](Golang/Golang基础/11_Go中的指针/README.md)
- [12_Go中的结构体](Golang/Golang基础/12_Go中的结构体/README.md)
- [13_Go中的包以及GoMod](Golang/Golang基础/13_Go中的包以及GoMod/README.md)
- [14_Go中的接口](Golang/Golang基础/14_Go中的接口/README.md)
- [15_goroutine实现并行和并发](Golang/Golang基础/15_goroutine实现并行和并发/README.md)
- [16_Golang中的反射](Golang/Golang基础/16_Golang中的反射/README.md)
- [17_Go中的文件和目录操作](Golang/Golang基础/17_Go中的文件和目录操作/README.md)
- [1_Go语言发展简史](Golang/Golang基础/1_Go语言发展简史/README.md)
- [2_Go的变量](Golang/Golang基础/2_Go的变量/README.md)
- [3_Go的数据类型](Golang/Golang基础/3_Go的数据类型/README.md)
- [4_Go的运算符](Golang/Golang基础/4_Go的运算符/README.md)
- [5_Go的流程控制](Golang/Golang基础/5_Go的流程控制/README.md)
- [6_Go的数组](Golang/Golang基础/6_Go的数组/README.md)
- [7_Go的切片](Golang/Golang基础/7_Go的切片/README.md)
- [8_Go的map](Golang/Golang基础/8_Go的map/README.md)
- [9_Go的函数](Golang/Golang基础/9_Go的函数/README.md)
- [CHANGELOG.md](Golang/Golang基础/Code/goProject/vendor/github.com/shopspring/decimal/CHANGELOG.md)
- [decimal](Golang/Golang基础/Code/goProject/vendor/github.com/shopspring/decimal/README.md)
- [gjson](Golang/Golang基础/Code/goProject/vendor/github.com/tidwall/gjson/README.md)
- [SYNTAX.md](Golang/Golang基础/Code/goProject/vendor/github.com/tidwall/gjson/SYNTAX.md)
- [match](Golang/Golang基础/Code/goProject/vendor/github.com/tidwall/match/README.md)
- [pretty](Golang/Golang基础/Code/goProject/vendor/github.com/tidwall/pretty/README.md)

## Golang 进阶

- [10_Flag包的用法](Golang/Golang进阶/10_Flag包的用法/README.md)
- [11_Go操作数据库](Golang/Golang进阶/11_Go操作数据库/README.md)
- [12_sqlx库的使用](Golang/Golang进阶/12_sqlx库的使用/README.md)
- [13_Go操作Redis](Golang/Golang进阶/13_Go操作Redis/README.md)
- [14_Go操作消息队列](Golang/Golang进阶/14_Go操作消息队列/README.md)
- [15_Go的依赖管理GoModule](Golang/Golang进阶/15_Go的依赖管理GoModule/README.md)
- [16_GoContext的使用](Golang/Golang进阶/16_GoContext的使用/README.md)
- [17_日志收集项目架构设计及Kafka介绍](Golang/Golang进阶/17_日志收集项目架构设计及Kafka介绍/README.md)
- [18_etcd介绍](Golang/Golang进阶/18_etcd介绍/README.md)
- [19_ES介绍和使用](Golang/Golang进阶/19_ES介绍和使用/README.md)
- [20_Kafka消费示例](Golang/Golang进阶/20_Kafka消费示例/README.md)
- [21_Kibana介绍和使用](Golang/Golang进阶/21_Kibana介绍和使用/README.md)
- [22_Prometheus和Grafana介绍](Golang/Golang进阶/22_Prometheus和Grafana介绍/README.md)
- [4_Golang并发编程](Golang/Golang进阶/4_Golang并发编程/README.md)
- [5_互联网协议介绍](Golang/Golang进阶/5_互联网协议介绍/README.md)
- [6_HTTP请求](Golang/Golang进阶/6_HTTP请求/README.md)
- [7_日志库](Golang/Golang进阶/7_日志库/README.md)
- [8_反射](Golang/Golang进阶/8_反射/README.md)
- [9_单元测试](Golang/Golang进阶/9_单元测试/README.md)

## Gin

- [1_Gin内容介绍](Golang/Gin框架/1_Gin内容介绍/README.md)
- [2_http及Template介绍](Golang/Gin框架/2_http及Template介绍/README.md)
- [3_GORM介绍和使用](Golang/Gin框架/3_GORM介绍和使用/README.md)
- [4_GORM进行CURD](Golang/Gin框架/4_GORM进行CURD/README.md)
- [Gin框架](Golang/Gin框架/README.md)

## Golang 其他

- [Golang](Golang/README.md)
- [1_go-zero架构设计](Golang/go-zero学习/1_go-zero架构设计/README.md)
- [2_go-zero介绍](Golang/go-zero学习/2_go-zero介绍/README.md)
- [3_go-zero适用场景](Golang/go-zero学习/3_go-zero适用场景/README.md)
- [4_使用goctl快速创建项目](Golang/go-zero学习/4_使用goctl快速创建项目/README.md)
- [5_使用goctl创建微服务项目](Golang/go-zero学习/5_使用goctl创建微服务项目/README.md)
- [1_RPC入门和创建中间件](Golang/grpc入门学习/1_RPC入门和创建中间件/README.md)
- [2_自签证书和服务器加入证书验证](Golang/grpc入门学习/2_自签证书和服务器加入证书验证/README.md)
- [0_序章](Golang/七天用Go实现Web框架-Gee/0_序章/README.md)
- [1_HTTP基础](Golang/七天用Go实现Web框架-Gee/1_HTTP基础/README.md)
- [3_上下文](Golang/七天用Go实现Web框架-Gee/3_上下文/README.md)
- [七天用Go实现Web框架-Gee](Golang/七天用Go实现Web框架-Gee/README.md)
- [Gin框架中让tmpl模板文件有语法提示](Golang/杂记/Gin框架中让tmpl模板文件有语法提示/README.md)
- [Goland添加注释模板](Golang/杂记/Goland添加注释模板/README.md)
- [Go项目分层结构规范](Golang/杂记/Go项目分层结构规范/README.md)
- [为什么有了go.mod还要用go.sum](Golang/杂记/为什么有了go.mod还要用go.sum/README.md)
- [国内环境执行GoGet命令失败的解决方法](Golang/杂记/国内环境执行GoGet命令失败的解决方法/README.md)

## 机器学习

- [CatBoost的安装和使用](机器学习/CatBoost的安装和使用/README.md)
- [Embedding层的作用](机器学习/Embedding层的作用/README.md)
- [Stacking算法介绍](机器学习/Stacking算法介绍/README.md)
- [使用SMOTE算法进行上采样](机器学习/使用SMOTE算法进行上采样/README.md)
- [1_决策树](机器学习/深入理解集成学习XGBoost及LightGBM/1_决策树/README.md)
- [2_集成学习](机器学习/深入理解集成学习XGBoost及LightGBM/2_集成学习/README.md)
- [深入理解集成学习XGBoost及LightGBM](机器学习/深入理解集成学习XGBoost及LightGBM/README.md)
- [集成学习](机器学习/集成学习/README.md)
- [Stacking](机器学习/集成学习/Stacking/README.md)
- [Youtube集成学习](机器学习/集成学习/Youtube集成学习/README.md)
- [机器学习算法之Boosting算法](机器学习/集成学习/机器学习算法之Boosting算法/README.md)
- [模型融合](机器学习/集成学习/模型融合/README.md)
- [金融风控问题案例](机器学习/集成学习/金融风控问题案例/README.md)

## 设计模式

- [设计模式](设计模式/设计模式/README.md)

## Python

- [reids](Python/reids/README.md)

## Rust

- [Rust](Rust/README.md)

## ES6

- [notes.md](ES6/notes.md)

## Ant Design

- [AntDesign](AntDesign/README.md)

## Umi

- [notes.md](Umi/notes.md)

## Tauri

- [tauri](tauri/README.md)

## 面试补充

- [Java中的专业词汇.md](面试八股文/Java中的专业词汇.md)
- [Spring源码解析](面试八股文/Spring源码解析/README.md)
- [集群高并发情况下如何保证分布式唯一全局Id生成](面试八股文/大厂面试第三季/集群高并发情况下如何保证分布式唯一全局Id生成/README.md)
