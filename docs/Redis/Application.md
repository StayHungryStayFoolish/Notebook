# Redis 应用

## 1. BloomFilter（布隆过滤器）

### 1.1 过滤器场景

- **问题**
- `如果在亿级数据过滤一条数据？`
    - 垃圾邮件识别
    - 拦截器
    - 防止缓存击穿
    - 去重（推荐不重复的商品、好友等）
- **方案**

    - **不推荐方案**
        - Set、Hashtable、HashMap 等占用空间大，效率低
    - **推荐方案**
        - BloomFilter**（概率型方案）**
- **简介**
  
  - [WIKI](https://en.wikipedia.org/wiki/Bloom_filter) ：A **Bloom filter** is a space-efficient [probabilistic](https://en.wikipedia.org/wiki/Probabilistic) [data structure](https://en.wikipedia.org/wiki/Data_structure), conceived by [Burton Howard Bloom](https://en.wikipedia.org/w/index.php?title=Burton_Howard_Bloom&action=edit&redlink=1) in 1970, that is used to test whether an [element](https://en.wikipedia.org/wiki/Element_(mathematics)) is a member of a [set](https://en.wikipedia.org/wiki/Set_(computer_science)). [False positive](https://en.wikipedia.org/wiki/Type_I_and_type_II_errors) matches are possible, but [false negatives](https://en.wikipedia.org/wiki/Type_I_and_type_II_errors) are not – in other words, a query returns either "possibly in set" or "definitely not in set." Elements can be added to the set, but not removed (though this can be addressed with the [counting Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter#Counting_Bloom_filters) variant); the more items added, the larger the probability of false positives.
  - **特点**
    - 一种空间效率高的 `概率型` 数据结构，用来检查一个元素是否在一个集合中
    - 对于一个元素检测是否存在的调用，BloomFilter 会返回两个结果之一：`False positive（可能存在）`或者`False negatives（一定不存在）`
- **原理**
  
  - `Bloom Filter` 有两个要素：长度为 **n** 的 `bit array` 和 **m** 个独立的 `hash function`，当要写入元素 `x` 的时候，用所有的 `hash function`  对 `x`  进行 `hash 后 mod n ` 得到 **m** 个位置，在 `bit array`  对应位置的 `bit` 设为 **1**，就完成了一次写入，验证元素是否存在，则只需要查看 `bit array` 上对应的 **m** 上的 `bit` 是否全部为 **1** 即可。 
  - 假设长度为 **10** 的 `bit array` 和 **3** 个 `hash function`，当写入元素 `element`，会分别计算 **3** 次 hash 并进行求模运算，然后获取 **3** 个 `index` 的值，最后在 `bit array` 对应的 index 将 `bit` 由 **0** 设置为 **1**。如下图所示
      - ![BloomFilter](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/BloomFilter.svg)
      - **优点**
          - 空间效率高，占用空间很小（可以参考 Bitmap）
          - 查询效率高
      - **缺点**
          - **误判率**
              - 如上图所示，当写入一个元素时，会操作 `bit array` 设置 **m（hash function 数量）** 个`bit` 为 **1**，当元素增多时，会有几率出现`不同元素之间的映射一致`，所以判断一个元素是否存在一个集合内的结果是 `False positive`，也就是会有误判率的情况存在
          - **删除困难**
              - 如上图所示，当一个 `bit array` 写入了元素后，很难进行删除，因为元素增多时，会有几率出现`不同元素之间的映射一致`。
  
### 1.1 Cuckoo Filter（升级版 - 布谷鸟过滤器）

-   [Cuckoo Filter](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf) 论文
    -   RedisBloom Modules 实现了 Cuckoo 论文，增加了计数、删除等功能

### RedisBloom Modules

-   [RedisBloom - Github](https://github.com/RedisBloom/RedisBloom)

-   [JRedisBloom](https://github.com/RedisBloom/JRedisBloom) Redis Labs - Java Library
  
-   （未实现布谷鸟命令，代码不多，感兴趣可以自己实现下）
  
- `Redis Bloom` 安装（实现了 Cuckoo Filter）

    - 本地安装

        - ```bash
            git clone https://github.com/RedisBloom/RedisBloom.git
            cd RedisBloom
            # 编译 会生成一个rebloom.so文件
            make 
            
            # 启动方式 1
            redis-server --loadmodule RedisBloom/rebloom.so
            # 启动方式 2  设置 INITIAL_SIZE 容量100万, 容错率 ERROR_RATE 0.0001, 占用空间是4m
            redis-server --loadmodule RedisBloom/rebloom.so INITIAL_SIZE 1000000   ERROR_RATE 0.0001   
            ```

    - Docker 安装

        - ```bash
            docker run -p 6379:6379 --name redis-redisbloom redislabs/rebloom:latest
            docker exec -it redis-redisbloom bash
            ```

    - **命令**

        - `Bloom Filter ` [BloomFilter Documentation](https://oss.redislabs.com/redisbloom/Bloom_Commands/)
            - `BF.RESERVE key error_rate capacity_size`
                - 初始化一个容量为 1000000 容错率为 0.001 的Bloom
                    -  `BF.RESERVE k1 0.001 1000000`
            - `BF.ADD key value`
            - `BF.EXISTS key value`
            - `BF.INSERT key CAPACITY count ERROR rate ITEMS value [value ...]`
                - 如果 k1 不存在则初始化容量为 1000，容错率为 0.001 的 Bloom，并添加了 v1、v2 两个元素
                    - `BF.INSERT k1 CAPACITY 1000 ERROR 0.001 ITEMS v1 v2`
            - `BF.INSERT key CAPACITY count ERROR rate NOCREATE ITEMS value [value ...]`
                - 只允许使用已有的 key 添加元素，不允许新建 key
                    - `BF.INSERT k1 CAPACITY 1000 ERROR 0.01 NOCREATE ITEMS v1 v2`
        - `Cuckoo Filter` 增加了删除命令 [CuckooFilter Documentation](https://oss.redislabs.com/redisbloom/Cuckoo_Commands/)
            - `CF.ADD key value`
            - `CF.EXISTS key value`
            - `CF.COUNT key value`
                - 查询 key 内元素 value 出现的次数
            - `CF.DEL key vaule`

### 1.3 Google Guava

-   [Google Guava - Github](https://github.com/google/guava) 实现了 `BloomFilter`

    -   ```java
        BloomFilter<String> bf = BloomFilter.create(Funnels.stringFunnel(Charsets.UTF_8), 1000, 0.0003);
        
        boolean result = bf.put("element"); // true
        
        boolean exist = bf.mightContain("element");  // true
        ```


## 2. Redis 实现 MQ 方案

### 2.1 基于 List 的 LPUSH + BRPOP

-   `LPUSH key value` 生产（向队列左侧压入）消息**（顺序消费）**
-   `RPUSH key value` 生产（向队列右侧压入）消息**（优先消费）**
-   `BRPOP key timeout` 消费（阻塞状态队列右侧弹出）消息
    -   **优点**
        -   实现简单
        -   消息可以持久化、保证顺序性
    -   **缺点**
        -   无法重复消费**（栈弹出模式）**
        -   不支持分组**（可以业务逻辑解决）**

### 2.2 基于 Sorted Set 使用 score（时间戳 + 序号） 保序

-   `ZADD key score member` **生产消息（使用 socre 进行排序）**
-   **消费消息**
    -   `ZRANGEBYSOCRE key min max WITHSCORES` **指定 score 最小范围（上次消费消息 ID 值）消费消息**
        -   `ZRANGEBYSCORE k1 -inf +inf WITHSCORES` 消费所有消息`（-inf 最小，+inf 最大）`，`-inf`在正常业务逻辑需要指定
    -   `ZRANGEBYSOCRE key min max WITHSCORES LIMIT offset count` 在消息最小、最大范围内，根据消息已有消息数量，`指定 offset` 读取一定数量的消息
        -   1.  `ZCARD key` 先获取数量（即 offset ）,假设是 **1000**
        -   2.  `ZRANGEBYSCORE k1 -inf +inf WITHSCORES LIMIT 1000 1`  根据偏移量 1000 消费新消息
    -   **优点**
        -   消息具有 ID 保序性
    -   **缺点**
        -   消息不能重复
        -   ID 必须保序性，不然漏读消息

### 2.3 基于 PUB / SUB 模式（PUSH 模式）

#### 2.3.1 订阅与发布模式，解耦生产者与消费者

-   `Redis` 的**发布与订阅**功能可以让客户端通过`广播方式`，将`消息（message）`同时发送给可能存在的多个客户端，并且发送消息的客户端不需要知道接收消息的客户端的具体信息,发布消息的客户端与接收消息的客户端两者之间没有直接联系。
    -   **优点**
        -   典型的广播模式
        -   主动 `PUSH` 模式，消费者自动接收订阅频道的信息
    -   **缺点**
        -   **消息无法存储**，订阅该频道的客户端不在线不会接收到消息，再次上线也不接收该时间点以前的消息
        -   **PUSH模式的弊端（生产 > 消费 ）**，客户端如果出现消息积压，达到一定程度会强制断开，导致消息丢失。
-   **PUB / SUB 三大组成部分**
    -   `频道（Channel）` 消息广播的通道 
    -   `发送者（Publisher）` 向通道发送消息的客户端
    -   `订阅者（Subscriber）` 订阅频道的客户端

![PUB/SUB](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/PUB_SUB.svg)

#### 2.3.2 订阅与发布模式基本命令

-   **订阅频道**
    -   `SUBSCRIBE channel [channel ...]`
    -   `PSUBSCRIBE parttern [parttern ...]` 模式匹配订阅频道
        -   `?` 匹配任意一个字符
        -   `*` 匹配多个字符
        -   `[]` 匹配其中任意一个字符
-   **向频道广播消息**
    -   `PUBLISH channel message`
-   **取消订阅**
    -   `UNSUBSCRIBE channel [channel ...]`
-   **查看订阅状态**
    -   查看被订阅频道
        -   `PUBSUB CHANNELS [pattern]` 
            -   不指定 `pattern`，返回所有客户端`已订阅的所有 channel 列表（只用 PUBLISH 没有被 SUBCRIBE 的不会统计）`
            -   指定 `pattern`，只返回匹配该模式的列表
    -   查看指定频道订阅数量
        -   `PUBSUB NUMSUB channel`
    -   查看匹配模式订阅频道数量
        -   `PUBSUB NUMPAT`

#### 2.3.3 订阅与发布底层原理

- **PUBSUB 结构体，所有订阅发布命令都是在操作 pubsub_channels 和 pubsub_patterns 两个结构体**

  - ```c
    struct redisServer {
      // ... 
      dict * pubsub_channels; // 字典，存储订阅 channel 的 clients（clinets 也是链表） 
      list * pubsub_patterns; // 链表，存储模式匹配的 client 和 匹配模式
      // ...
    }
    
    // 使用 PSUBSCRIBE 模式匹配命令订阅的 client 存储在这个结构体
    typedef struct pubsubPattern {
      client *client;       	// 订阅该模式匹配的 client
      robj *pattern;          // 模式匹配结构
    } pubsubPattern;
    ```

  - `pubsub_channels` 

    - 字典结构，`key` 存储 `channel` 的值，`value` 存储订阅该 `channel` 的所有的 `clinets 链表`。

  - `pubsub_patterns`

    - 链表结构，每个节点存储 `pubsubPattern` 结构体。

- **发布消息 PUBLISH channel message**

- `PUBLISH` 命令执行完毕之后会同步到 `Redis从服务`。这样，如果一个客户端 `订阅了从服务的 channel`，在主服务中向该 channel 推送消息时，该客户端也能收到推送的消息。

  1. 从 `pubsub_channels` 字典中以 `channel` 为 key，取出所有订阅了该 `channel` 的 `client`，依次向每个客户端 `PUSH` 数据。
  2. 依次遍历 `pubsub_patterns`，将链表中每个节点的模式匹配结构 `pattern` 与推送的 `channel` 进行匹配，如果匹配成功，说明该节点的 `client` 订阅了模式匹配的 `channel`， 向该客户端`PUSH` 数据。

- **订阅频道 SUBSCRIBE channel [ channel ... ]**

- 当一个客户端执行 SUBSCRIBE 命令后会进入`PUB/SUB` 模式，在该种模式下，该客户端只能执行如下几类命令：PING、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE 和 PUNSUBSCRIBE。

  1. `SUBSCRIBE` 会先去 `pubsub_channels` 字典中以 `channel` 为 key 查看是否有此次订阅该 `channel` 的 `client`，如果有直接返回。
  2. 如果没有，将此次订阅的 `channel` 作为 key 存储在 `pubsub_channels` 字典中，并将 `client` 作为一个元素是存储在 key 对应的 `clients 值链表中`。
  3. 进入 `PUB/SUB` 模式。

- **匹配模式订阅 PSUBSCRIBE**

  - 针对 `pubsub_pattern` 链表操作，逻辑同 `SUBSCRIBE`。

- **取消订阅 UNSUBSCRIBE**

  - `SUBSCRIBE` 操作的结构体反向操作。

- **匹配模式取消订阅 PUNSUBCRIBE**

  - `PSUBCRIBE` 反向操作。

### 2.4 Stream 模式

[Stream](http://notebook.bonismo.ink/#/Redis/DataType?id=_9-stream)

## 3. Locks（锁）

### 3.1 单 JVM 内的锁

[Oracle 关于 Locks的解释](https://docs.oracle.com/cd/E13150_01/jrockit_jvm/jrockit/geninfo/diagnos/thread_basics.html)：When threads in a process share and update the same data, their activities must be synchronized to avoid errors. In Java, this is done with the `synchronized` keyword, or with `wait` and `notify`. Synchronization is achieved by the use of locks, each of which is associated with an object by the JVM. For a thread to work on an object, it must have control over the lock associated with it, it must “hold” the lock. Only one thread can hold a lock at a time. If a thread tries to take a lock that is already held by another thread, then it must wait until the lock is released. When this happens, there is so called “contention” for the lock.

-   关于 Locks 大意如下：
    -   锁是避免多个线程同时操作一个资源发生错误，通过锁由 `JVM` 与一个对象（资源）关联，保证在一个时间节点上，只有一个线程在该对象上工作，从而保证数据的一致性。如果另外一个线程视图获取该线程已持有的锁，则必须等到该线程释放，同一把锁在 **同一个 JVM 内** 具有`互斥性`。
-   根据以上定义可以确定，当一个对象在一个时间节点上只有一个 `JVM` 的线程执行时，可以使用 `JVM 内部的 synchronized` 或者 `java.util.concurrent.locks.Lock`，使用场景只能是 `单体应用单机部署`，随着业务的扩展，单体应用并发访问量上不去的时候，需要水平扩展改为 `单体应用集群部署`，这种场景下，上述两种锁就会失效，因为已经 `跨 JVM` 了，架构演变到现在的 `微服务`和 `Serverless` 也同样不适用。

### 3.2 分布式锁

分布式锁解决的是单机部署的锁控制策略失效问题。解决该问题则需要一种 `跨 JVM 的互斥机制` 控制一个时间节点上对一个对象的访问。


