---
title: SLF4J及日志栈溢出问题
date: 2018-05-04 14:47:34
tags:
    - Java
    - SLF4J
---

新接手一项目，在本地测试运行时，发现报如下错误:
```Java
    Detected both log4j-over-slf4j.jar AND slf4j-log4j12.jar on the class path, preempting StackOverflowError.
```
很明显，是项目的classpath中同时存在log4j-over-slf4j.jar和slf4j-log4j12.jar，引起死循环。但为什么项目在线上能好好运行呢？
原来线上运行的程序在打包时做了手脚，在最终的依赖包中，删除了log4j和slf4j-log4j12，如下：
```Java
    <dependencySets>
        <dependencySet>
            <outputDirectory>lib</outputDirectory>
            <excludes>
                <exclude>log4j:log4j</exclude>
                <exclude>org.slf4j:slf4j-log4j12</exclude>
            </excludes>
            <useProjectArtifact>false</useProjectArtifact>
        </dependencySet>
    </dependencySets>
```

让我们来重头捋一下Java日志框架及该问题的成因。

我们用的最多的几个日志框架有Log4j、Jakarta Commons Logging以及Logback-classic。尤其值得一提的是Logback-classic，它是Log4j作者写的又一个开源日志组件，由于实现了SLF4J's的Logger接口，native，性能优越，可以直接作为SLF4J的实现。

[SLF4J](https://www.slf4j.org/index.html)，全称Simple Logging Facade for Java，是一个抽象的日志框架，支持程序动态绑定日志框架实现，如绑定到`java.util.logging`、`logback`以及 `log4j`等，使用时需要在应用中添加依赖slf4j-api。

**SLF4J对应用日志做了封装抽象，因此，业务可以针对SLF4J进行编程，再按需绑定具体的日志框架。无论是灵活性还是可扩展性，都有很多的提升。**

为了支持动态绑定不同的日志框架，SLF4J提供了几个绑定工具，如下：

| 绑定 | 说明 |
| ----|---- |
| **slf4j-log4j12** | Log4j绑定，Log4j是一个使用非常广泛的日志组件|
| **slf4j-jcl** | Jakarta Commons Logging绑定|
| **slf4j-jdk14** | java.util.logging, JDK绑定|
| **slf4j-simple** | 输出会定向到System.err，只有INFO级别以上的才会输出|
| **slf4j-nop** | 丢弃日志绑定|

为了方便jcl、log4j、jul用户迁移到SLF4J，SLF4J提供了几个桥接模块。

| 模块 | 目标对象 | 使用说明 | 实现原理 | 
| :---- | :---- | :---- | :---- | 
| **log4j-over-slf4j** | log4j用户 | 将log4j.jar替换为log4j-over-slf4j.jar|直接替换了log4j中的大量同名类，如Logger、Category、Level等|
| **jcl-over-slf4j** | jcl用户 | 将commons-logging.jar替换为jcl-over-slf4j.jar |保留了jcl的接口，但底层实现采用了slf4j|
| **jul-to-slf4j** | jul用户 | 直接依赖 | 由于jul在java.*namespace下，无法替换。该模块采用了翻译手段，将LogRecord转换为slf4j类似对象。** 该方案存在一定的性能损失，不推荐使用** |

按上面的介绍，使用slf4j时就需要注意一下几个问题：
1. **`jcl-over-slf4j.jar`和`slf4j-jcl.jar`不能同时依赖！**
    - 原因：jcl会将实现委托到slf4j，而slf4j又会绑定到jcl。
    
2. **`log4j-over-slf4j.jar`和`slf4j-log4j12.jar`不能同时依赖！**
    - 原因: log4j会将实现委托到slf4j，而slf4j又会绑定到log4j实现。
    
3. **`jul-to-slf4j.jar`和`slf4j-jdk14.jar`不能同时依赖！**
    - 原因：jul会将实现委托到slf4j，而slf4j又会绑定到jdk实现。
    
**上述三个情况下，若同时依赖均会产生循环依赖问题，导致StackOverflowError!**

回到开头的问题，即上述第2个注意问题，怎么解决也已经很明显了。

**参考资料**
1. [SLF4J user manual](https://www.slf4j.org/manual.html)
2. [Bridging legacy APIs](https://www.slf4j.org/legacy.html)