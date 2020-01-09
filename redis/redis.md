## Redis特点

- 高性能
  - 操作基于内存
  - io多路复用
  - 单进程 无进程切换，同步
  - 数据结构简单 没有关系模式(相比mysql)
- 原子性
  - 单进程



## 数据结构与对象

### 字符串

- 字符串字面量 string literal
  - 用于无需对字符串值进行修改的地方，比如打印日志
  - redisLog(REDIS_WARNING, "hhhhhh");(redisLog应该是个redis的内部函数)
- 可变字符串 simple dynamic string SDS
  - set msg "hello world" 创建一个键值对
    - 键是一个sds，保存着msg
    - 值也是一个sds，保存"hello world"

  - 实现

    ![](../pic/sds结构体alloc.png)

    ![](../pic/sds实例.png)

    - alloc表示buf数组的大小(**不包括字符串最后的空字符**)
    - len是字符串长度——已使用的空间，也就是实际保存的字符串的长度
    - free表示已分配未使用的空间，**源码中没有这个属性**
    - 所以 free = alloc - len
    - 那么结构体的真实大小是 结构体头的大小 + alloc + 1
    - 而且字符数组的空间与结构体的空间是连续的，因为分配内存的时候，同时分配了结构体和数组的空间，所以可以通过结构体的地址与结构体的大小相加得到数组的位置，这样的话buf属性其实没有作用(**事实上在源码中也确实没被使用到**，除了在**object.createEmbeddedStringObject()**中使用到了)
  - buf存储字符串的字符数组，已使用字符的最后一位保存空字符'\0'，使用了**柔性数组**
      - sds遵循c字符串以空字符结尾的规则，须为空字符额外分配1字节空间，且**不计入len**
      - 空字符的操作由sds函数自动完成，对使用者完全透明
      - 这样的好处是，可以直接重用一部分c字符串函数库中的函数
  
  - 与c的字符串的区别
  
    - 常熟复杂度获取字符串长度
      - 结构体中保存有长度，O1时间可以获取
      - 而c语言的字符串需要遍历一遍字符
    - 杜绝缓冲区溢出
      - c语言对字符串的操作可能出现缓冲区溢出
      - 例如strcat在字符串A后面拼接一个新的字符串B，C语言会直接拼接；假如A的长度小于拼接后的长度，那么多出的部分就会覆盖掉A后面的字节，出现缓冲区溢出
      - 而SDS在每次操作之前都会确保空间满足修改要求，不满足的话API会将空间扩展至执行修改所需的空间大小(如何扩展看下一点)，所以SDS不需要手动修改空间大小，也不会出现缓冲区溢出
    - 减少修改字符串时带来的内存重新分配次数
      - c字符串不记录自身长度以及分配的内存大小，所以对于一个n的字符的c字符串，总是一个n+1个字符的数组(因为有空字符)，而每次增长或者缩短字符串，都需要重新分配内存，不然会缓冲区溢出或者内存泄露
      - redis经常被用于对速度要求严苛、数据被频繁修改的场合，如果每次修改字符串的长度都要执行一次内存分配，那么内存分配的时间会占去修改字符串所用时间的一大部分
      - SDS的实现
        - 通过"未使用空间free"，解除了字符串长度和底层数组长度的关联，buf的长度不一定等于字符串长度+1，因为可能包含未使用字节
        - SDS的优化策略
          - 空间预分配——当SDS需要空间**扩展**时，不仅会分配修改所需的必要的空间，还会分配额外的未使用空间
            - 对SDS修改后，SDS长度小于1MB，那么程序分配和len相同的free；例如修改之后字符串长度len变为13，那么free也会是13，那么buf的实际长度就是13+13+1
            - 如果修改之后SDS的len大于等于1MB，那么free会变为1MB
          - 惰性空间释放——当SDS缩短时，程序不会立即使用内存重分配来回收内存，而是使用free属性来将多出来的字节记录下来，并等待将来使用(那么什么时候释放呢)
            - 将来需要扩展的时候可以直接使用这部分多出来的空间
            - 空余空间需要手动释放sdsRemoveFreeSpace，会重新分配空间
          - 二进制安全
            - c字符串以\0为字符结束，所以无法存放二进制数据，因无法保证二进制数据中不出现\0的数据
            - sds的所有api都以二进制的方式来处理

### hash

### list

插入顺序有序

### set

插入顺序无序

### sortedSet/zset

有序set，按zset xxszet  nscore asdafa 中的score排序，权重

## 命令

- keys pattern 返回所有符合给定模式的key
  - 一次性返回所有
  - key过多会卡住
- scan cursor MATCH pattern COUNT n
  - 扫描cursor开始的匹配pattern的key 数量大概n个
  - cursor 必须有
    - 第一次scan cursor为0
    - 每次返回数据包括新的cursor，当返回cursor为0时，代表scan结束
  - count不一定满足 

## 分布式锁

### 设计要点

- 互斥
- 安全
  -  不能由其他没持有锁的客户端删除
- 死锁
  - 假如客户端宕机，使得其他客户端无法获取锁
- 容错
  - 当某些redis宕机时仍然可以获取锁

### 使用

- setnx key value 设置如果不存在
  - 如果key不存在 set成功返回1 否则失败 返回0
  - 问题
    - 无法失效
    - 需要调用expire
    - 非原子性
  - 解决办法
    - redis 2.6.12以后支持
    - set key value EX seconds/PX milliseconds NX/XX
      - EX seconds 秒
      - PX 毫秒
      - NX 不存在时set
      - XX 存在时set
    - 成功返回ok
    - 失败返回nil

### 大量key同时过期

设置过期时加随机值

### Redis做异步队列

- 实现

  - 使用list
  - rpush 生成
  - lpop 消费

- 问题

  - 不会等待有数据然后消费，没数据会直接返回

- 解决

  - lpop无数据，然后重试

  - blpop key timeout 阻塞等待有数据

- 只能单消费者

  - 可以使用发布订阅模式

### 发布订阅

- subscribe xxtopic
- publish xxtopic value
- 问题
  - 无状态，不保证到达
  - 假如消费者下线，然后上线期间发布的消息读不到

### 持久化

- rdb/快照
  - 某个时间点全量数据的快照
  - 配置
    - save 900 1 900秒1条写入就持久化
    - save 300 10 
    - save 60 10000
    - rdbcompress rdb压缩
  - 命令
    - save 阻塞主进程持久化
    - bgsave 后台持久化
      - copyOnWirte
  - 发生时机
  - 缺点
    - 持久化全量文件
    - 定时持久化，会丢失
- aof 增量
  - 配置
    - 默认 appendfsync everysec 每秒写入
  - 每次把客户端命令append到aof文件
  - aof会有重复命令 比如1incr10次，就可以直接set 11
  - 改进 aofrewrite
    - fork一个子进程
    - 子进程赋值一份新的aof写到临时文件，不依赖原来的aof文件
    - 主进程持续把新的命令写到内存和原来的aof文件
    - 主进程获取子进程重写完成的信号，往新的aof中同步增量变动
    - 用新的aof替换旧的aof
- 混合持久化
  - 先全量rdb
  - 增量aof
- 数据恢复
  - 先aof
  - 后rdb

### pipeline批量执行命令

无需等待上一个命令结果

- 返回成功失败命令数

### 同步

- 全同步
  - slave发送sync命令给master
  - master bgsave
  - 增量命令缓存起来
  - bgsave完成后发送给slave
  - slave载入rdb文件
  - master将增量命令发送给slave
- 部分重同步/psync 2.8以上
  - 由以下几个部分实现
    - mater的复制偏移量offset
    - mater的复制积压缓冲区
    - master的runid
  - 实现
    - slave
      - 如果slave之前没slaveOf过master
        - 不需要发送runid offset
        - 进行全同步
      - 否则发送runid和offset
    - master
      - runid不一样或者offset超过了master积压缓冲区
        - 返回全同步，runid，offset
      - 否则
        - 返回continue 将进行部分同步

### 哨兵 sentinel

- 监控master slave状态
- 故障通知
- 自动主从切换
- 协议
  - 流言协议
  - 最终一致

### 集群

- 缓存命中问题，希望每个机器缓存某些key

- 一致hash
  - key对2^32取模
  - 顺时针寻找node
  - node宕机、新增 只有node之前的数值受影响
- 问题
  - 数据倾斜，缓存不均匀
    - 添加虚拟节点A-1，A-2，A-3，hash到A的虚拟节点的数据，都由A存储



