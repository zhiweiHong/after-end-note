# `Redis` 相关知识点整理

## 为什么选择`redis`

- JAVA实现的Map是本地缓存，如果有多台机器的话，每个实例都需要各自保存一份缓存，缓存不具备<u>一致性</u>。
- `Redis`实现的是分布式缓存，如果是多台机器的话，每个实例都共享一份缓存，缓存具有一致性。
- JAVA实现的Map不是专业做缓存的，没有过期时间的说法。而且`JVM`内存过大很容易挂掉。

## `Redis`数据结构

常用的有`string`、`list`、`hash`、`set`、`sortset`这几种。

## `Redis`存储相关

`Redis`使用对象来表示数据库中的键和值。每次我们在`Redis`数据库中新创建一个键值对时，**至少会创建出两个对象**。一个是键对象，一个是值对象。

## `Redis` 服务器中的数据库

`Redis`也有数据库的概念，如果不指定数量的话，默认是16个数据库。不同数据库的数据隔离的，也就是说1号数据库无法访问到2号数据库相关的数据。

![img](https://mmbiz.qpic.cn/mmbiz_png/2BGWl1qPxib0LSf7wiaom7XfZ9RhpUWsW2aVhiajoicKWmwOOM5icLpTiaBWQJRFD2oT5dMPtXWp14TreuWF5ibTVRydg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 键的过期时间

`Redis`是基于内存，而内存是有限的。所以需要干掉不常用的数据，保留常用的数据。这就需要我们设置一下过期时间。

- 设置键的生存时间，可以通过`expire`或者`pexpire`命令。
- 设置键的过期时间，可以通过`expireat`或者`pexpireat`命令

其实，`expire`、`pexpire`和`expireat`都是通过`pexpireat`实现的

既然有设置过期(生存)时间的命令，那肯定也有移除过期时间，查看剩余生存时间的命令了：

- `PERSIST`(移除过期时间)
- `TTL`(Time To Live)返回剩余生存时间，以秒为单位
- `PTTL`以毫秒为单位返回键的剩余生存时间

### 过期策略

- 定时删除（对内存友好，对CPU不友好）
  - 到时间点上把所有过期的键值都删除了
- 惰性删除（对CPU极度优化，对内存极度不友好）
  - 每次从键值空间取值的时候，判断一下该键是否过期了，如果过期了就删除。
- 定期删除（折中）
  - 每个一段时间去删除过期键，限制删除的执行时长和频率

`Redis`采用的是惰性删除+定期删除两种策略，所以说，在`Redis`里边如果过期键到了过期的时间了，未必被立马删除的！

## 内存淘汰机制

- `volatile-lru` 
  - 从已设置过期时间的数据集中挑选最少使用的数据删除
- `volatile-ttl`
  - 从已设置过期时间的数据集中挑选将要过期的数据淘汰
- `volatile-random`
  - 从已设置过期时间的数据集中任意选择数据淘汰
- `allkeys-lru`
  - 从所有数据集中挑选最近使用的数据淘汰
- `allkeys-random`
  - 从所有数据集中任意选择数据淘汰
- `noeviction`
  - 禁止驱逐数据

使用 `Redis` 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是**热点数据**。可以将内存最大使用量设置为热点数据占用的内存量，然后启用`allkeys-lru`淘汰策略，将最近最少使用的数据淘汰

## `Redis`持久化

### `RDB（快照持久化）`

快照持久化可以手动执行，也可以根据服务器配置的定期执行，`RDB`持久化所生成的`RDB`文件是经过压缩的二进制文件，`Redis`可以通过这个文件还原数据库。

两个命令可以生成`RDB`文件

- SAVE会阻塞`Redis`服务进程，服务器无法接受任何请求，直到`RDB`文件创建完毕为止
- `BGSAVE`，background save，顾名思义，后台保存。由子进程来负责创建`RDB`文件，服务器进程可以继续接受请求。

`Redis`服务启动的时候，如果发现有`RDB`文件，会自动载入`RDB`文件。服务器在载入`RDB`文件期间，会处于阻塞状态，直到载入工作完成。

#### 对待过期键的策略

- 执行`SAVE`或者`BGSAVE`命令创建出的`RDB`文件，程序会对数据库中的过期键检查，**已过期的键不会保存在`RDB`文件中**。
- 载入`RDB`文件时，程序同样会对`RDB`文件中的键进行检查，**过期的键会被忽略**。

### `AOF`（append only File）

`AOF`是通过保存`Redis`服务器执行的写命令来记录数据的。

比如说我们对空白的数据库执行以下写命令：

```
redis> SET meg "hello"
OK

redis> SADD fruits "apple" "banana" "cherry"
(integer) 3

redis> RPUSH numbers 128 256 512
(integer) 3 
```

`Redis`会产生以下内容的`AOF`文件：

![AOF文件](https://mmbiz.qpic.cn/mmbiz_png/2BGWl1qPxib0LSf7wiaom7XfZ9RhpUWsW2qK9ugpcc95LvEiaiam3QfoaPS0ibGfD1ZCkUoSJDWwGiaG39rxadZrLTyw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

`AOF`持久化功能的实现可以分为3个步骤：

- 命令追加：命令写入`aof_buf`缓冲区
- 文件写入：调用`flushAppendOnlyFile`函数，考虑是否要将`aof_buf`缓冲区的写入`AOF`文件中
- 文件同步：考虑是否将内存缓冲区的数据真正的写入到硬盘中

![img](https://mmbiz.qpic.cn/mmbiz_png/2BGWl1qPxib0LSf7wiaom7XfZ9RhpUWsW2MjkXbZETIPbC23Br1ZrMNFaQmfU94kicv1H9vLNIU0VzMPFNnRdbRSw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### `AOF`重写

当日志达到一定的程度的时候（默认是64M）会触发AOF的日志重写。

**AOF重写不需要对现有的AOF文件进行任何的读取、分析。AOF重写是通过读取服务器当前数据库的数据来实现的**！

可以通过配置文件，也可以通过`BGREWRITEAOF`手动触发。

AOF配置重写触发条件：

![image-20200602150603335](C:\Users\zhiwei hong\AppData\Roaming\Typora\typora-user-images\image-20200602150603335.png)

#### 对待过期键的策略

- 如果数据库的键已过期，但还没被惰性/定期删除，`AOF`文件不会因为这个过期键产生任何影响(也就说会保留)，当过期的键被删除了以后，会追加一条DEL命令来显示记录该键被删除了
- 重写`AOF`文件时，程序会对`RDB`文件中的键进行检查，**过期的键会被忽略**。

## `Redis`单线程为什么快？

- 纯内存操作

- 核心是基于非阻塞的IO多路复用机制
- 单线程避免了多线程的频繁上下文切换问题

## 缓存雪崩

### 什么是缓存雪崩？

- `redis`挂掉了，所有的请求全部走数据库
- 对缓存数据设置相同的过期时间，导致某段时间内缓存失败，请求全部走数据库。

![å¦æç¼å­ææäºï¼å¨é¨è¯·æ±è·å»æ°æ®åºäº](https://user-gold-cdn.xitu.io/2019/1/11/1683aabdeb81d181?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 解决办法

- 针对于所有的key值设置同一个时间
  - 在缓存的时候给过期时间加上一个**随机值**，这样就会大幅度的**减少缓存在同一时间过期**。
- 针对于`Redis`挂掉了
  - 实现`Redis`的高可用
  - 如果`Redis`全部挂了，则使用本地缓存+限流的方式

## 缓存穿透

#### 什么事缓存穿透？

缓存穿透是指查询一个一定**不存在的数据**。由于缓存不命中，并且出于容错考虑，如果从**数据库查不到数据则不写入缓存**，这将导致这个不存在的数据**每次请求都要到数据库去查询**，失去了缓存的意义。

#### 解决方法

- 由于请求的参数是不合法的(每次都请求不存在的参数)，于是我们可以使用布隆过滤器(BloomFilter)或者压缩filter**提前拦截**，不合法就不让这个请求到数据库层！
- 当我们从数据库找不到的时候，我们也将这个**空对象设置到缓存里边去**。下次再请求的时候，就可以从缓存里边获取了。
  - 这种情况我们一般会将空对象设置一个**较短的过期时间**。