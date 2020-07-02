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

#### 订阅与发布模式基本命令

-   订阅频道
    -   `SUBSCRIBE channel [channel ...]`
-   向频道广播消息
    -   `PUBLISH channel message`
-   取消订阅
    -   `UNSUBSCRIBE channel [channel ...]`
-   查看信息
    -   查看被订阅频道
        -   `PUBSUB CHANNELS `
    -   查看频道订阅数量
        -   `PUBSUB NUMSUB`
    -   查看被订阅模式总数量
        -   `PUBSUB NUMPAT`

#### 订阅与发布模式图示

-   `Redis` 的**发布与订阅**功能可以让客户端通过`广播方式`，将`消息（message）`同时发送给可能存在的多个客户端，并且发送消息的客户端不需要知道接收消息的客户端的具体信息,发布消息的客户端与接收消息的客户端两者之间没有直接联系。
-   `频道（Channel）` 消息广播的通道 
-   `发送者（Publisher）` 向通道发送消息的客户端
-   `订阅者（Subscriber）` 订阅频道的客户端

![PUB/SUB](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/PUB_SUB.svg)



