---
title: JVM调优及问题排查
date: 2020-07-29 17:15:15
categories: 
- Java
tags: 
- JVM
---

## 前言

本文是针对有一定Java虚拟机基础知识后，在实际工作中解决JVM相关问题及如何进行优化，偏方法论。

## Java内存区域及收集器介绍

<center><font color="red">jdk1.7及之前</font></center>

![](https://pic.downk.cc/item/5f2146e814195aa59465fe0a.png)

<center><font color="red">jdk1.8</font></center>

![](https://pic.downk.cc/item/5f2146e814195aa59465fe0a.png)

1.8同1.7比，最大的差别就是：**元数据区取代了永久代**。元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：**元数据空间并不在虚拟机中，而是使用本地内存**。

![](https://pic.downk.cc/item/5f222aa914195aa594e49ad8.png)

垃圾收集器：

收集算法、垃圾收集器详细介绍：略，待补充。

垃圾收集从2个方面不断的改进：提高垃圾回收效率；提高用户体验（低停顿）；

![](https://pic.downk.cc/item/5f222c2314195aa594e52e60.png)

<center>HotSpot虚拟机的垃圾收集器（介绍见参考一）</center>

## JVM调优目标

- 一、何时需要做jvm调优？

  ​	1、heap 内存（老年代）持续上涨达到设置的最大内存值；

  ​	2、Full GC 次数频繁；

  ​	3、GC 停顿时间过长（超过1秒）；

  ​	4、应用出现OutOfMemory 等内存异常；

  ​	5、应用中有使用本地缓存且占用大量内存空间；

  ​	6、系统吞吐量与响应性能不高或下降。

- 二、 JVM调优原则

  ​	1、多数的Java应用不需要在服务器上进行JVM优化；

  ​	2、多数导致GC问题的Java应用，都不是因为我们参数设置错误，而是代码问题；

  ​	3、在应用上线之前，先考虑将机器的JVM参数设置到最优（最适合）；

  ​	4、减少创建对象的数量；

  ​	5、减少使用全局变量和大对象；

  ​	6、JVM优化是到最后不得已才采用的手段；

  ​	7、在实际使用中，分析GC情况优化代码比优化JVM参数更好；

- 三、 JVM调优目标

  ​	1、GC低停顿；

  ​	2、GC低频率；

  ​	3、低内存占用；

  ​	4、高吞吐量;

## JVM调优参数简介

#### 堆设置

| 参数                        | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| -Xmn1200m                   | 设置年轻代大小为1200MB。增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8 |
| -Xms4g                      | 初始化堆内存大小为4GB                                        |
| -Xmx4g                      | 堆内存最大值为4GB                                            |
| -Xss512k                    | 设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1MB，以前每个线程堆栈大小为256K。应根据应用线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右 |
| -XX:MaxDirectMemorySize     | 堆外内存/直接内存的大小，默认为堆内存减去一个Survivor区的大小 |
| -XX:MaxMetaspaceSize=512m   | 设置元数据区最大值512M（<font color="red">jdk1.8</font>）    |
| -XX:MaxPermSize=256m        | 设置持久代大小为256MB。（<font color="red">jdk1.7及以下</font>） |
| -XX:MaxTenuringThreshold=15 | 设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论 |
| -XX:MetaspaceSize=128m      | 设置元数据区初始值128M（<font color="red">jdk1.8</font>）    |
| -XX:NewRatio=4              | 设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5 |
| -XX:PermSize=100m           | 初始化永久代大小为100MB。（<font color="red">jdk1.7及以下</font>） |
| -XX:ReservedCodeCacheSize   | JIT编译后二进制代码的存放区，满了之后就不再编译。默认开多层编译240M，可以在JMX里看看CodeCache的大小 |
| -XX:SurvivorRatio=8         | 设置年轻代中Eden区与Survivor区的大小比值。设置为8，则两个Survivor区与一个Eden区的比值为2:8，一个Survivor区占整个年轻代的1/10 |

#### GC设置

| 参数                                     | 说明                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| -XX:+HeapDumpOnOutOfMemoryError          | 发生内存溢出是进行heap-dump                                  |
| -XX:HeapDumpPath=/path/to/java_pid.hprof | 这个参数与-XX:+HeapDumpOnOutOfMemoryError共同作用，设置heap-dump时内容输出文件 |
| -XX:ErrorFile=/path/to/hs_err_pid.log    | 指定致命错误日志位置。一般在JVM发生致命错误时会输出类似hs_err_pid.log的文件，默认是在工作目录中（如果没有权限，会尝试在/tmp中创建），不过还是自己指定位置更好一些，便于收集和查找，避免丢失 |
| -XX:StringTableSize=1000003              | 指定字符串常量池大小，默认值是60013。对Java稍微有点常识的应该知道，字符串是常量，创建之后就不可修改了，这些常量所在的地方叫做字符串常量池。如果自己系统中有很多字符串的操作，且这些字符串值比较固定，在允许的情况下，可以适当调大一些池子大小 |

#### GC日志

GC过程可以通过GC日志来提供优化依据。

| 参数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| -XX:+PrintGCDetails                                          | 启用gc日志打印功能                                           |
| -Xloggc:/path/to/gc.log                                      | 指定gc日志位置                                               |
| -XX:+PrintHeapAtGC                                           | 打印GC前后的详细堆栈信息                                     |
| -XX:+PrintGCDateStamps                                       | 打印可读的日期而不是时间戳                                   |
| -XX:+PrintGCApplicationStoppedTime                           | 打印所有引起JVM停顿时间，如果真的发现了一些不知什么的停顿，再临时加上-XX:+PrintSafepointStatistics -XX: PrintSafepointStatisticsCount=1找原因 |
| -XX:+PrintGCApplicationConcurrentTime                        | 打印JVM在两次停顿之间正常运行时间，与-XX:+PrintGCApplicationStoppedTime一起使用效果更佳 |
| -XX:+PrintTenuringDistribution                               | 查看每次minor GC后新的存活周期的阈值                         |
| -XX:+UseGCLogFileRotation 与 -XX:NumberOfGCLogFiles=10 与 -XX:GCLogFileSize=10M | GC日志在重启之后会清空，但是如果一个应用长时间不重启，那GC日志会增加，所以添加这3个参数，是GC日志滚动写入文件，但是如果重启，可能名字会出现混乱 |
| -XX:PrintFLSStatistics=1                                     | 打印每次GC前后内存碎片的统计信息                             |

#### 收集器

| 参数                          | 说明1                                                        | 说明2                                                        |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -XX:+UseConcMarkSweepGC       | 设置[CMS收集器](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#concurrent_mark_sweep_cms_collector) （只针对老年代） | 并发收集、低停顿；对CPU资源敏感，CMS默认启动的回收线程数是（CPU数量+3）/4； |
| -XX:+UseParNewGC              | 设置年轻代为多线程收集                                       | 可以不设置，jdk 8中设置-XX:+UseConcMarkSweepGC，自动启用-XX:+UseParNewGC |
| -XX:+CMSClassUnloadingEnabled | 配合-XX:+UseConcMarkSweepGC使用，垃圾回收会清理持久代，移除不再使用的classes |                                                              |
| -XX:+ UseG1GC                 | 允许使用垃圾优先（G1）垃圾收集器。它是一个服务器式垃圾收集器，针对具有大量RAM的多处理器计算机。它以高概率满足GC暂停时间目标，同时保持良好的吞吐量。 | G1收集器推荐用于需要大堆（大小约为6 GB或更大）且GC延迟要求有限的应用（稳定且可预测的暂停时间低于0.5秒）。 |
| G1其他参数详情请见参考一      |                                                              |                                                              |

#### 其他参数设置

| 参数                                     | 说明                                                         |
| ---------------------------------------- | :----------------------------------------------------------- |
| -ea                                      | 启用断言，这个没有什么好说的，可以选择启用，或这选择不启用，没有什么大的差异。完全根据自己的系统进行处理 |
| -XX:+UseThreadPriorities                 | 启用线程优先级，主要是因为我们可以给予周期性任务更低的优先级，以避免干扰客户端工作。在我当前的环境中，是默认启用的 |
| -XX:ThreadPriorityPolicy=42              | 允许降低线程优先级                                           |
| -XX:+HeapDumpOnOutOfMemoryError          | 发生内存溢出是进行heap-dump                                  |
| -XX:HeapDumpPath=/path/to/java_pid.hprof | 这个参数与-XX:+HeapDumpOnOutOfMemoryError共同作用，设置heap-dump时内容输出文件 |
| -XX:ErrorFile=/path/to/hs_err_pid.log    | 指定致命错误日志位置。一般在JVM发生致命错误时会输出类似hs_err_pid.log的文件，默认是在工作目录中（如果没有权限，会尝试在/tmp中创建），不过还是自己指定位置更好一些，便于收集和查找，避免丢失 |
| -XX:StringTableSize=1000003              | 指定字符串常量池大小，默认值是60013。对Java稍微有点常识的应该知道，字符串是常量，创建之后就不可修改了，这些常量所在的地方叫做字符串常量池。如果自己系统中有很多字符串的操作，且这些字符串值比较固定，在允许的情况下，可以适当调大一些池子大小 |
| -XX:+AlwaysPreTouch                      | 在启动时把所有参数定义的内存全部捋一遍。使用这个参数可能会使启动变慢，但是在后面内存使用过程中会更快。可以保证内存页面连续分配，新生代晋升时不会因为申请内存页面使GC停顿加长。通常只有在内存大于32G的时候才会有感觉 |
| -XX:-UseBiasedLocking                    | 禁用偏向锁（在存在大量锁对象的创建且高度并发的环境下(即非多线程高并发应用)禁用偏向锁能够带来一定的性能优化） |
| -XX:AutoBoxCacheMax=20000                | 增加数字对象自动装箱的范围，JDK默认-128～127的int和long，超出范围就会即时创建对象，所以，增加范围可以提高性能，但是也是需要测试 |
| -XX:-OmitStackTraceInFastThrow           | 不忽略重复异常的栈，这是JDK的优化，大量重复的JDK异常不再打印其StackTrace。但是如果系统是长时间不重启的系统，在同一个地方跑了N多次异常，结果就被JDK忽略了，那岂不是查看日志的时候就看不到具体的StackTrace，那还怎么调试，所以还是关了的好 |
| -XX:+PerfDisableSharedMem                | 启用标准内存使用。JVM控制分为标准或共享内存，区别在于一个是在JVM内存中，一个是生成/tmp/hsperfdata_{userid}/{pid}文件，存储统计数据，通过mmap映射到内存中，别的进程可以通过文件访问内容。通过这个参数，可以禁止JVM写在文件中写统计数据，代价就是jps、jstat这些命令用不了了，只能通过jmx获取数据。但是在问题排查是，jps、jstat这些小工具是很好用的，比jmx这种很重的东西好用很多，所以需要自己取舍。这里有个GC停顿的例子 |
| -Djava.net.preferIPv4Stack=true          | 这个参数是属于网络问题的一个参数，可以根据需要设置。在某些开启ipv6的机器中，通过InetAddress.getLocalHost().getHostName()可以获取完整的机器名，但是在ipv4的机器中，可能通过这个方法获取的机器名不完整，可以通过这个参数来获取完整机器名 |

## JVM如何调优

#### 步骤

​	第1步：分析GC日志（详见参考四、五）及dump文件，判断是否需要优化，确定瓶颈问题点；

​    第2步：确定JVM调优量化目标；

​	第3步：确定JVM调优参数（根据历史JVM参数来调整）；

​	第4步：调优一台服务器，对比观察调优前后的差异；

​	第5步：不断的分析和调整，直到找到合适的JVM参数配置；

​	第6步：找到最合适的参数，将这些参数应用到所有服务器，并进行后续跟踪。

#### 命令

jps：查询java进程

jstat：虚拟机运行状态信息

jmap：生成堆存储快照

jstack：生成虚拟机当前时刻的线程快照，帮助定位线程出现长时间停顿的原因

参考：https://www.jianshu.com/p/c6a04c88900a

## 实例

不同的环境有不同的方案，可以参考，不过还是建议在自己的环境中有针对的验证之后再使用，毕竟各自的环境都有差异。

#### gc查看

查看gc.log状态，可参考 <font color="red">参考四、参考五</font>；

结论：没有Full GC；每次从年轻代移到了老年代的内存占老年代总内存比例并不大：

![](https://pic.downk.cc/item/5f223ba014195aa594eca7aa.png)

查看star.hprof文件（使用JProfiler），如果有的话；

查询某一时刻虚拟机运行状态；

./jdk/jdk1.8.0_101/bin/jstat -gc 【pid】 3600s

| 参数 | 说明                              | 设置的值           |
| ---- | --------------------------------- | ------------------ |
| S0C  | Survivor0空间的大小。单位KB。     | 316992.0（0.3G）   |
| S1C  | Survivor1空间的大小。单位KB。     | 316992.0（0.3G）   |
| S0U  | Survivor0已用空间的大小。单位KB。 | 0                  |
| S1U  | Survivor1已用空间的大小。单位KB。 | 17521.7            |
| EC   | Eden空间的大小。单位KB。          | 2536320.0（2.4G）  |
| EU   | Eden已用空间的大小。单位KB。      |                    |
| OC   | 老年代空间的大小。单位KB。        | 3121152.0（3G）    |
| OU   | 老年代已用空间的大小。单位KB。    | 221391（0.21G）    |
| MC   | 方法区大小                        | 125824.0（0.12G）  |
| MU   | 方法区使用大小                    | 119041（0.11G）    |
| CCSC | 压缩类空间大小                    | 13824.0（0.01G）   |
| CCSU | 压缩类空间使用大小                | 12536              |
| YGC  | 年轻代垃圾回收次数                | 261                |
| YGCT | 年轻代垃圾回收消耗时间            | 10.931（0.04s/次） |
| FGC  | 老年代垃圾回收次数                | 0                  |
| FGCT | 老年代垃圾回收消耗时间            | 0                  |
| GCT  | 垃圾回收消耗总时间                | 10.931             |

#### 查看系统jvm参数状态

机器状况

| 内存 | CPU  | 磁盘空间 |
| ---- | ---- | -------- |
| 8G   | 4*4  | 60G      |

当前jvm参数配置：

```jvm
-Xms6144m -Xmx6144m -XX:MaxPermSize=256m -verbose:gc -XX:+PrintGCDetails -Xloggc:/../tomcat/logs/gc.log -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m -Xmn3096m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:+ExplicitGCInvokesConcurrent -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/../tomcat/logs/star.hprof 
```

|                                                              |                                                              |                                                              |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 参数                                                         | 说明                                                         | 建议                                                         |
| -Xms6144m                                                    | 初始化堆内存大小为6GB                                        | 低于系统内存                                                 |
| -Xmx6144m                                                    | 堆内存最大值为6GB                                            | 低于系统内存                                                 |
| ~~-XX:MaxPermSize=256m~~                                     | 设置永久代最大为256MB（无用）                                | jdk1.8不需要设置                                             |
| -verbose:gc                                                  | 输出虚拟机中GC的详细情况                                     |                                                              |
| -XX:+PrintGCDetails                                          | 输出虚拟机中GC的详细情况                                     |                                                              |
| -Xloggc:/../tomcat/logs/gc.log                               | 设置gc日志文件位子                                           |                                                              |
| -XX:+PrintGCTimeStamps                                       | 打印gc时间时间戳，gc.log                                     |                                                              |
| -XX:MetaspaceSize=256m                                       | 设置元数据区初始值256M （永久代）                            | 该值越大触发Full GC的时机就越晚，                            |
| -XX:MaxMetaspaceSize=256m                                    | 设置元数据区最大值256M （永久代）                            |                                                              |
| -Xmn3096m                                                    | 设置年轻代区域3G                                             | Sun官方推荐配置为整个堆的3/8                                 |
| -XX:+UseConcMarkSweepGC                                      | 设置<font color="red">[CMS收集器](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#concurrent_mark_sweep_cms_collector) （只针对老年代）</font> | 并发收集、低停顿；对CPU资源敏感，CMS默认启动的回收线程数是（CPU数量+3）/4； |
| ~~-XX:+UseParNewGC~~                                         | 设置<font color="red">年轻代为多线程收集</font>              | 可以不设置，jdk 8中设置-XX:+UseConcMarkSweepGC，自动启用-XX:+UseParNewGC |
| -XX:+CMSClassUnloadingEnabled                                | 配合-XX:+UseConcMarkSweepGC使用，垃圾回收会清理持久代，移除不再使用的classes |                                                              |
| -XX:+ExplicitGCInvokesConcurrent                             | 允许使用`System.gc()`请求调用并发GC 。默认情况下禁用此选项，并且只能与该`-XX:+UseConcMarkSweepGC`选项一起启用 |                                                              |
| -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses            | 通过`System.gc()`在并发GC周期期间使用请求和卸载类，可以调用并发GC。默认情况下禁用此选项，并且只能与该`-XX:+UseConcMarkSweepGC`选项一起启用 |                                                              |
| -XX:+HeapDumpOnOutOfMemoryError                              | 通过使用堆分析器（HPROF）将Java堆转储到当前目录中的文件      |                                                              |
| -XX:HeapDumpPath=/home/ewallet/loan-star/loan-star-pre/default/tomcat/logs/star.hprof | 设置用于写入堆分析器（HPROF）提供的堆转储的路径和文件名。默认情况下，该文件在当前工作目录中创建，并且名为java_pidpid.hprof，其中pid是导致错误的进程的标识符 |                                                              |

#### 总结

当前机器的配置足够，在目前情况下优化参数不会有太大的提升效果。

假如在降低机器配置前提下，那需要依赖数据，做相应的调整，具体要看机器成本而定。

## 参考

#### 附一 找到占用cpu最高的一个线程

- 第一步，找到占用cpu最高的一个线程pid

  方法：直接top,然后shift+h

- 第二步，将其转化成16进制。假使我们得到的线程号为【n】，接下来将它转成16进制，记为spid

  方法：printf 0x%x 【n】

- 第三步，执行jstack -l 【pid】| grep 【spid】 -A 100 打印后面100行分析问题

 插件使用：https://github.com/oldratlee/useful-scripts

#### 附二 参考资料

参考一：[https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html ](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)（ <font color="red">参数</font>）

​       https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html（<font color="red">收集器</font>）

​       https://www.oracle.com/technetwork/java/tuning-139912.html（<font color="red">调优试例</font>）

参考二：书籍：深入理解Java虚拟机（第二版） 

​       书籍：Java Performance：The Definitive Guide

 参考三：https://blog.csdn.net/linhu007/article/details/48897597（<font color="red">CMS与G1</font>）

参考四：https://blog.csdn.net/zc19921215/article/details/83029952

参考五：https://www.aliyun.com/jiaocheng/1341282.html