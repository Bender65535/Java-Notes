# Redis面试题

### redis是什么

Redis是C语言开发的一个开源的（遵从BSD协议）高性能键值对（key-value）的内存数据库，可以用作数据库、缓存、消息中间件等。它是一种NoSQL（not-only sql，泛指非关系型数据库）的数据库。 

Redis作为一个内存数据库。  性能优秀，数据在内存中，读写速度非常快 ， 单进程单线程，是线程安全的，采用IO多路复用机制； 

丰富的数据类型，支持字符串（strings）、散列（hashes）、列表（lists）、集合（sets）、有序集合（sorted sets）等； 

支持数据持久化。可以将内存中数据保存在磁盘中，重启时加载； 

支持 主从复制，哨兵，高可用； 

可以用作分布式锁；  可以作为消息中间件使用，支持发布订阅 

### redis 和 memcached 的区别

1. **redis支持更丰富的数据类型（支持更复杂的应用场景）**：Redis不仅仅支持简单的k/v类型的数据，同时还提供list，set，zset，hash等数据结构的存储。memcache支持简单的数据类型，String。
2. **Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用,而Memecache把数据全部存在内存之中。**
3. **集群模式**：memcached没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；但是 redis 目前是原生支持 cluster 模式的.
4. **Memcached是多线程，非阻塞IO复用的网络模型；Redis使用单线程的多路 IO 复用模型。**

### redis 为什么是单线程的？

 因为 cpu 不是 Redis 的瓶颈，Redis 的瓶颈最有可能是机器内存或者网络带宽。既然单线程容易实现，而且 cpu 又不会成为瓶颈，那就顺理成章地采用单线程的方案了。 可以避免多线程上下文切换。



### 为什么redis这么快？

完全基于内存,绝大部分请求是纯粹的内存操作,执行效率高
采用单线程,单线程也能处理高并发请求,想多核也可启动多实例

单线程反而避免了多线程的频繁上下文切换问题，预防了多线程可能产生的竞争问题。 

核心是基于非阻塞的 IO 多路复用机制。 

### redis 支持的数据类型有哪些？应用？

string、list、hash、set、zset。

**String 在你们项目怎么用的？**

 **常用命令:** set,get,decr,incr,mget 等。 

在显示某个人的基本数据的时候，比如名字，粉丝数，关注数，使用 String 保存：

``` 
eg:  user:id:3506728370  
{"id":3506728370,"name":"春晚","fans":12210862,"blogs":6164, "focus":83}
```

设置一个定时刷新的操作，这样用户不需要直接读取数据库。怎么设置？setx key   value，一定时间循环判断key是否失效，到期后再去数据库读取。

**List 在你们项目怎么用的？**

 **常用命令:** lpush,rpush,lpop,rpop,lrange等 

1. 朋友圈点赞，要求按照点赞顺序显示点赞好友信息
   如果取消点赞，移除对应好友信息，但是不能使用pop了，怎么办呢？

   解决方案

   lrem key count value **移除指定数据**
   count：移除的数目
   value：具体要移除的内容

2. 个人用户的关注列表需要按照用户的关注顺序展示。

**Set 在你们项目怎么用的？**

每位用户首次使用今日头条时会设置3项爱好的内容，但是后期为了增加用户的活跃度、兴趣点，必须让用户
对其他信息类别逐渐产生兴趣，增加客户留存度，如何实现？

**分析**
系统分析出各个分类的最新或最热点信息条目并组织成set集合
随机挑选其中部分信息
配合用户关注信息分类中的热点信息组织成展示的全信息集合

**解决方案**

- 随机获取集合中指定数量的数据	

  srandmember key [count]

- 随机获取集合中的某个数据并将该数据移出集合

  spop key [count]

**zset**

  在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息，适合使用 Redis 中的 Sorted Set 结构进行存储。 

### redis 设置过期时间

Redis中有个设置时间过期的功能，即对存储在 redis 数据库中的值可以设置一个过期时间。作为一个缓存数据库，这是非常实用的。如我们一般项目中的 token 或者一些登录信息，尤其是**短信验证码都是有时间限制的**，按照传统的数据库处理方式，一般都是自己判断过期，这样无疑会严重影响项目性能。

我们 set key 的时候，都可以给一个 expire time，就是过期时间，通过过期时间我们可以指定这个 key 可以存活的时间。

如果假设你设置了一批 key 只能存活1个小时，那么接下来1小时后，redis是怎么对这批key进行删除的？

**定期删除+惰性删除。**

通过名字大概就能猜出这两个删除方式的意思了。

- **定期删除**：redis默认是每隔 100ms 就**随机抽取**一些设置了过期时间的key，检查其是否过期，如果过期就删除。注意这里是随机抽取的。为什么要随机呢？你想一想假如 redis 存了几十万个 key ，每隔100ms就遍历所有的设置过期时间的 key 的话，就会给 CPU 带来很大的负载！
- **惰性删除** ：定期删除可能会导致很多过期 key 到了时间并没有被删除掉。所以就有了惰性删除。假如你的过期 key，靠定期删除没有被删除掉，还停留在内存里，除非你的系统去查一下那个 key，才会被redis给删除掉。

但是仅仅通过设置过期时间还是有问题的。我们想一下：如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期key堆积在内存里，导致redis内存块耗尽了。怎么解决这个问题呢？ **redis 内存淘汰机制。**

### 数据淘汰机制

当内存到达最大内存限制时进行的数据淘汰策略

**配置**

最大可用内存：maxmemory //默认为0，一般设置全部内存50%以上
每次选取带删除数据个数：maxmemory-samples //采用随机获取方式
删除策略：maxmemory-policy //达到最大内存后，对被选取带数据进行的删除策略

**检测易失数据集（ **设置了过期时间的键空间）
volatile-lru：挑选最近最少使用的数据淘汰（最近数据中使用时间离当前最远的数据）。**常用**
volatile-lfu：挑选最近使用次数最少的数据淘汰（最近数据中使用次数最少的数据）
volatile-ttl：挑选将要过期数据淘汰
volatile-random：任意挑选数据淘汰

**检测全库数据（所有数据集）**
allkeys-lru：挑选最近最少使用的数据淘汰
allkeys-lfu：挑选最近使用次数最少的数据淘汰
allkeys-random：任意挑选数据淘汰

**放弃数据驱逐**

no-enviction //禁止驱逐数据 4.0中默认策略，会引发OOM

**LRU 算法实现**：1.通过双向链表来实现，新数据插入到链表头部；2.每当缓存命中（即缓存
数据被访问），则将数据移到链表头部；3.当链表满的时候，将链表尾部的数据丢弃。
LinkedHashMap：HashMap 和双向链表合二为一即是 LinkedHashMap。HashMap 是无序
的，LinkedHashMap 通过维护一个额外的双向链表保证了迭代顺序。该迭代顺序可以是插
入顺序（默认），也可以是访问顺序。



### redis 持久化的两种方式

- RDB：RDB 持久化机制，是对 redis 中的数据执行**周期性**的持久化。
- AOF：AOF 机制对每条写入命令作为日志，以 `append-only` 的模式写入一个日志文件中，在 redis 重启的时候，可以通过**回放** AOF 日志中的写入指令来重新构建整个数据集。

通过 RDB 或 AOF，都可以将 redis 内存中的数据给持久化到磁盘上面来，然后可以将这些数据备份到别的地方去，比如说阿里云等云服务。

如果 redis 挂了，服务器上的内存和磁盘上的数据都丢了，可以从云服务上拷贝回来之前的数据，放到指定的目录中，然后重新启动 redis，redis 就会自动根据持久化数据文件中的数据，去恢复内存中的数据，继续对外提供服务。

如果同时使用 RDB 和 AOF 两种持久化机制，那么在 redis 重启的时候，会使用 **AOF** 来重新构建数据，因为 AOF 中的**数据更加完整**。

**RDB 持久化优点**
RDB是一个紧凑压缩的二进制文件，存储效率高
RDB恢复数据速度比AOF快 

 **RDB持久化缺点**
无法做到实时持久化，具有较大可能丢失数据
存储数量较大时，效率较低，I／O性能较低
基于fork创建子进程，内存产生额外消耗
宕机带来的数据丢失风险 

**AOF 优点**

- AOF 可以更好的保护 数据不丢失，一般 AOF 会每隔 1 秒，  最多丢失 1 秒钟的数据。 
-  写入性能非常高，而且文件不容易破损 
-  **适合做灾难性的误删除的紧急恢复**。 

**AOF 缺点**

 对于同一份数据来说，AOF 日志文件通常比 RDB 数据快照文件更大。 

 恢复速度较慢 

#### RDB 与 AOF 如何选择

对数据非常敏感，建议使用默认的AOF持久化方案
AOF策略使用everysec，每秒fsync一次，该策略仍可保持很好性能，出现问题最多丢失一秒内的数据
数据呈现阶段有效性，建议使用RDB持久化方案
数据可以做到阶段内无丢失，且恢复较快，阶段点数据恢复通常使用RDB方案

注意：
AOF文件存储体积较大，恢复速度较慢
利用RDB使用线紧凑的数据持久化会使Redis性能降低

综合：
RDB与AOF选择实际上是在一种权衡，每种都有利有弊
如果不能承受分钟内的数据丢失，对业务数据非常敏感，选用AOF
如果能承受分钟内的数据丢失，且追求大数据集的恢复速度选用RDB
灾难恢复选用RDB
双保险策略，同时开启RDB和AOF，重启后Redis优先使用AOF来恢复数据，降低丢失数据量



### 项目中缓存是如何使用的？

这个，需要结合自己项目的业务来。

### 为什么要用缓存？

用缓存，主要有两个用途：**高性能**、**高并发**。



### 怎么保证缓存和数据库数据的一致性？

- 合理设置缓存的过期时间。
- 新增、更改、删除数据库操作时同步更新 Redis，可以使用事务机制来保证数据的一致性。

### redis 怎么实现分布式锁？

Redis 分布式锁其实就是在系统里面占一个“坑”，其他程序也要占“坑”的时候，占用成功了就可以继续执行，失败了就只能放弃或稍后重试。

占坑一般使用 setnx(set if not exists)指令，只允许被一个程序占有，使用完调用 del 释放锁

也可以配合`EXPIRE key seconds`自动释放锁
设置key的生存时间,当key过期时(生存时间为0) ,会被自动删除
风险/ **缺陷** ：原子性没有得到满足，所以不建议。

### 缓存雪崩



**在一个较短的时间内，缓存中较多的key集中过期或者缓存挂了**，导致了**数据库服务器崩溃**

缓存雪崩的事前事中事后的解决方案如下：

在批量往Redis存数据的时候，把每个Key的失效时间都加个随机值就好了，这样可以保证数据不会再同一时间大面积失效。 

### 缓存穿透

 **原因：** 

1. Redis中大面积出现未命中
2. 出现非正常URL访问

 解决方案：最简单粗暴的方法如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们就把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。 

 **布隆过滤器（Bloom Filter）**这个也能很好的预防缓存穿透的发生，他的原理也很简单，就是利用高效的数据结构和算法快速判断出你这个Key是否在数据库中存在，不存在你return就好了，存在你就去查DB刷新KV再return 

###  缓存击穿

缓存击穿是指一个Key非常热点，在不停地扛着大量的请求，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发直接落到了数据库上，就在这个Key的点上击穿了缓存。 

解决：设置热点数据永不过期，或者加上互斥锁就搞定了。

**假如 Redis  里面有 1  亿个 key ，其中有 10w 个 个 key  是以某个固定的已知的前缀开头的，如**
**果将它们全部找出来？**
使用 keys 指令可以扫出指定模式的 key 列表。
对方接着追问：如果这个 redis 正在给线上的业务提供服务，那使用 keys 指令会有什么问
题？
这个时候你要回答 redis 关键的一个特性：redis 的单线程的。keys 指令会导致线程阻塞一
段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用 scan 指
令，scan 指令可以无阻塞的提取出指定模式的 key 列表，但是会有一定的重复概率，在客
户端做一次去重就可以了，但是整体所花费的时间会比直接用 keys 指令长。

 
