## 常用参数

- `-Xms` 最小堆内存 `-Xmx` 最大堆内存

  - **-Xms256M 直接写大小不加等号 不加空格**

- `-XX:NewSize` 新生代大小 `-XX:MaxNewSize` 最大新生代大小

- `-XX:NewRatio` 新生代和年老代比例 `-XX:SurvivorRatio` survivor区和eden区比例

- `-XX:MetaspaceSize` metaspace大小 `-XX:MaxMetaSpaceSize` 最大metaspace大小

  - > https://www.cnblogs.com/williamjie/p/9558136.html

  - 如果没有配置`-XX:MetaspaceSize`，那么触发FGC的阈值是21807104（约20.8m），可以通过jinfo -flag MetaspaceSize pid得到这个值；

  - 如果配置了`-XX:MetaspaceSize`，那么触发FGC的阈值就是配置的值；

  - Metaspace由于使用不断扩容到`-XX:MetaspaceSize`参数指定的量，就会发生FGC；且之后每次Metaspace扩容都会发生FGC；

  - 如果Old区配置CMS垃圾回收，那么扩容引起的FGC也会使用CMS算法进行回收；

  - 如果`MaxMetaspaceSize`设置太小，可能会导致频繁FGC，甚至OOM；

  - **JDK8+移除了Perm，引入了Metapsace，它们两者的区别是什么呢？Metasace上面已经总结了，无论`-XX:MetaspaceSize`和`-XX:MaxMetaspaceSize`两个参数如何设置，随着类加载越来越多不断扩容调整，直到MetaspaceSize(如果没有配置就是默认20.8m)触发FGC，上限是`-XX:MaxMetaspaceSize`，默认是几乎无穷大。而Perm的话，我们通过配置`-XX:PermSize`以及`-XX:MaxPermSize`来控制这块内存的大小，jvm在启动的时候会根据`-XX:PermSize`初始化分配一块连续的内存块，这样的话，如果`-XX:PermSize`设置过大，就是一种赤果果的浪费。很明显，Metapsace比Perm好多了^^；**

    **JDK7 Perm 验证如下--设置`-XX:PermSize=64m -XX:MaxPermSize=64m`，那么PC初始化就是64m：**

- `-XX:+UseCompressedClassPoiters` 是否启用压缩类指针

- `-XX:CompressedClassSpaceSize` 压缩类指针空间大小

## 命令

- jps 查看java进程
- jstack pid 查看线程信息
  - 包含线程状态
  - 最后会有死锁提醒
- jmap pid 查看内存映像
  - -dump 导出后可以mat分析
- jinfo pid 查看jvm参数
  - -flag xxx(见参数部分)
  - -flags 查看所有可用参数
- jstat pid
  - -gc 查看gc情况
- jhat dump文件 (.hprof)
  - 好丑好丑。。。。