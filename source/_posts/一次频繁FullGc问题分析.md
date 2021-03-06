title: 一次频繁FullGc问题分析
date: 2018-05-29 23:32:48
tags: [Java]
categories: Java
---
## 问题背景
应公司Pigeon服务组RD要求，升级维护的老项目的pigeon版本，结果升级成功后系统频繁FullGc报警

```
监控项:   sum(#5) jvm.fullgc.count  > 5
当前值: 2
时间: 2018-05-29 15:07:00
```
```
监控项:   all(#1) jvm.memory.oldgen.used.percent.after.fullgc  >= 60
当前值: 77.97195
```
## 问题分析
就升级依赖的基础服务的版本，不至于出问题吧，真是见了鬼了。首先想到的是pigeon的问题，wiki搜了FAQ，没有相关问题；问了相关负责人，有没有升级版本出现FullGc的问题，回复是没有。所以只能猜想是服务本身的问题。回滚pigeon版本，重启服务，观察GC日志，发现还是频繁fullgc,确认是老版本的pigeon没有添加相关的监控。所以只能是服务本身出了问题。

通过top命令发现线上机器内存大小为8G，tomcat bin目录下的setenv.sh设置的JVM参数如下

```
JVM_MEM_OPTS="-server -Xmx5g -Xms5g -XX:PermSize=1g -XX:MaxPermSize=800m -XX:SurvivorRatio=8 -XX:+HeapDumpOnOutOfMemoryError -XX:ReservedCodeCacheSize=128m -XX:InitialCodeCacheSize=128m -XX:NewSize=4g -XX:MaxNewSize=4g"

JVM_GC_OPTS="-XX:+DisableExplicitGC -XX:+UseParallelGC -XX:ParallelGCThreads=4 -XX:+UseParallelOldGC -XX:+UseAdaptiveSizePolicy -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCApplicationConcurrentTime -XX:+PrintGCApplicationStoppedTime -XX:+PrintHeapAtGC -XX:+PrintTenuringDistribution -XX:+PrintCommandLineFlags -XX:ParallelCMSThreads=4 -XX:+CMSClassUnloadingEnabled -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=10 -XX:CMSInitiatingOccupancyFraction=80 -Xloggc:/tmp/logs/gc/pig.gc.log -XX:ErrorFile=/tmp/logs/gc/pig.vmerr.log -XX:HeapDumpPath=/tmp/logs/gc/pig.heaperr.log"
```
堆大小给了5G，新生代就给了4G，也就是老年代只有1G，猜想是老年代太小了，因为新生代minor GC的时候，如果存活对象比较多，很快就把老年代填满，因此很容易fullgc。因为是老服务，不确定之前这样设置新生代是否合理，因此增大堆大小为6G。观察发现，效果不明显。缩小新生代大小为3G，增大老年代为3G后观察，minor GC次数增多，fullGc次数减少，报警明显减少。
使用jstat观察gc情况，发现gc数目还是简直丧心病狂

```
$ jstat -gc 715  250 20
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT
314368.0 314368.0  0.0   876.1  2516992.0 1473400.4 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473406.6 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473408.8 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473595.4 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473595.4 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473608.1 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473608.1 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473608.1 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473620.2 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473620.4 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473620.4 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473620.4 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473807.7 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473831.3 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473856.8 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473858.8 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473858.8 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473907.5 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473907.5 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
314368.0 314368.0  0.0   876.1  2516992.0 1473907.5 3145728.0   564697.4  512000.0 137860.4  14745 10408.377 5914  9404.969 19813.345
```

没办法了，只能在fullgc之前把堆dump下来，在jvm参数中添加+XX:HeapDumpBeforeFullGC,用eclipse mat分析dump日志，
如下图所示：
![heapdum](/images/heapdump.png)
查看引用的对象
![reference object](/images/large_result_set.png)
对应到代码中，发现sql对应的代码是使用rabbitmq同步人员／部门的基础数据相关，同步之前会sql查询出来数据做一些数据补充和校验，猜想是这个导致了fullgc的问题

由于mq同步是每天凌晨批量触发，观察线上falcon监控
![fullgc](/images/fullgc.png)
![younggc](/images/younggc.png)
![cpu_busy](/images/cpu_busy.png)
发现时间刚好吻合

按道理来说rabbitmq是单线程消费，分配的对象应该是在egen区才对，这些对象在一次消费完成后就不再适用，在egen填满后minor gc的时候应该都能回收，就不会进入老年代中，导致fullgc。问题根源还是在于，大对象是直接进入老年代的。至此，问题的根源已经清楚。
## 参考
+ [触发JVM进行Full GC的情况及应对策略](https://blog.csdn.net/chenleixing/article/details/46706039>)