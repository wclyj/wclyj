# 使用 DevTools 实现 SpringBoot 项目热部署

## 前言

本地同时改好几个进程时，每次保存都整站重启，等待时间会叠在一起。DevTools 可以在 classpath 变化后只重启应用上下文，改完类很快看到结果。

## 热部署是什么

改一行业务代码，想看到效果，通常要重新编译出 class，再由类加载器读进虚拟机。热重启沿用这个事实：发现 class 变了，就丢掉旧的应用类加载器，换一个新的去加载，Spring 上下文跟着重建。它不是替换方法体的热替换，静态字段和单例都会重新创建。

## 类加载机制

编译器把源码变成 class 文件。虚拟机把文件中的字节读进内存，校验、解析、初始化，最后得到能直接使用的类型。DevTools 用两个加载器把这个过程拆开：基础加载器负责第三方依赖，重启加载器负责自己的 `target/classes`。依赖没变就不必重读，所以第二次启动比冷启动短。

## 添加依赖

坐标是 `org.springframework.boot:spring-boot-devtools`。Gradle 写法：

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-devtools"
}
```

Maven 里给这项依赖加上 `optional 元素（值为 true）`，避免打成可执行包时把重启能力带到生产。Gradle 则用 `developmentOnly`，效果相同。

## DevTools 的配置

```yaml
spring:
  devtools:
    restart:
      enabled: true
      additional-paths: src/main/java
      exclude: static/**,public/**
```

`additional-paths` 让目录变化也触发重启。`exclude` 里的静态资源只刷新资源，不重建上下文。

## IDEA 需要的配置

DevTools 看的是 classpath 上的文件时间。IDEA 默认在程序跑着的时候不自动编译，所以要打开：

- Settings → Compiler → Build project automatically
- 按 `Ctrl+Shift+Alt+/` 打开 Registry，勾上 `compiler.automake.allow.when.app.running`

保存 Java 文件后应看到一次 restart 日志。只改了 `exclude` 里的页面时，不应出现整站重启。
