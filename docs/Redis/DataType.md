# Redis 的 9 种数据类型

## Redis 数据类型

**每种数据类型介绍前，会有 1 - 3 行的黑体介绍，快速认识当前数据类型。**

### 基本数据类型

-   String
-   Hash
-   List
-   Set
-   Sorted Set

### 高级数据类型

-   Bitmap
-   HyperLogLog
-   Geohash 
-   Stream

## 1.String

>   进入Redis 客户端使用  help @string 查看命令
>
>   示例：https://redis.io/commands#string

**String 最基本的键值对数据类型，一种 key 和 value 可以是 String，也可以是普通的文字、图片、视频等二进制数据**

### 1.1 二进制安全

-   计算机存储的最小单位是 `Bit（比特，也成位）`，一个`Byte（字节）`。1 Byte = 8 bit。每个 Bit 用 `0` 或 `1` 来表示。`Character（字符）`通过不同的字符编码（ASCII、Unicode、UTF-8 、GBK等）由其指定固定的字节来表示。
-   `二进制安全` 是一个`输入`和`输出`以`字节`为单位的流，当数据`输入`时，不会对数据进行任何`限制`、`过滤`等（*例如：C 语言使用长度 N + 1 的方式表示字符串，N 为字符串，1 表示字符数组最后一个元素以'\0' 结尾，当读取到以 '\0' 结尾时结束，如果字符串本身就包含'\0'字符，则读取字符串会被截断，即非二进制安全的。*）。即无论`字符`以任何编码形式，最终存储的只是`字节`，保证了`输入`时的原始数据。存储和读取的双方只需约定好编码集就可以获取数据内容，具有跨平台、跨语言、防止数据类型溢出等优点。**所以读取双方需要事先约定好编码格式。**

### 1.2 Redis 的 SDS（Simple Dynamic Strings）

-   C 语言的 `char`存储方式

    -   ![C char](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/c-char.png)
        -   以 `\0` 结尾，当获取`char`的`length`需要遍历数组直到空字符（**\0**）停止，复杂度为 **0(N)**
        -   `char`本身不记录长度，容易产生缓冲区溢出。

-   **SDS 结构及特性**

    ```c
    typedef char *sds;
    
    struct sdshdr {
        // buf 已占用长度，占用 4 个字节（64位系统）
        int len;
        // buf 剩余可用长度，占用 4 个字节（64位系统）
        int free;
        // 实际保存字符串数据的地方
        char buf[];
    };
    
    // 例如存储 hello world 为例
    struct sdshdr {
        // buf 已占用长度
    	len = 11;
        // buf 剩余可用长度
    	free = 0;
        // buf 实际长度为 len + 1 = 12
    	buf = “hello world\0”;
    };
    ```

    -   `len` 和 `free` 成为头部。方便获取字符串长度。
    -   **内容存在在 buf [ ] 数组中。SDS 既然是字符串，则首先需要一个字符串指针，方便上层接口调用。SDS 对上层暴露的指针不是只想结构体 SDS 的指针，而是直接指向柔型数组 buf [ ] 的指针。这样可以直接读取实际存储的内容，并兼容 C 语言处理字符串的各种函数。**
    -   由于有长度有统计变量 `len`，所以读取字符串时不依赖 **'\0'** 终止符，保证了二进制安全。
    -   通过 `len` 属性， `sdshdr` 可以实现复杂度为 **θ(1)** 的长度计算及读操作。
    -   **SDS 为了更少的占用空间，在 Redis 不断迭代的版本中一直在优化，深入了解可以查阅相关资料，这里简单说下 sdsdr5，头部占用空间变小，只负责存储小于 32 字节的字符串，SDS 会根据一定条件进行扩容为 sdsdr8。**


### 1.3 字符串操作

#### 1.3.1 String 常用命令

```bash
# 设置 key 和 value 并可以设置过期时间的原子操作
SET key value [expiration EX seconds|PX milliseconds] [NX|XX]

# 获取 key 的 value
GET key

# 追加 value 到原来 value 的最后（如果 key 不存在会创建空字符串并追加）
APPEND key value

# 设置一个带过期时间的 key 和 value ，单位：秒
SETEX key seconds value

# 设置一个带过期时间的 key 和 value ，单位：毫秒
PSETEX key milliseconds value

# 获取指定索引范围内的字符串值
GETRANGE key start end

# 获取字符串值的字节的长度
STRLEN key
```

-   **SET 命令会强制覆盖 key 所持有的 value，无视 value  任何数据类型都会被强制覆盖**
  
    -   **官方文档：**Set `key` to hold the string `value`. If `key` already holds a value, it is overwritten, regardless of its type. Any previous time to live associated with the key is discarded on successful [SET](https://redis.io/commands/set) operation.
    
-   **STRLEN** 的字节长度有编码集有关，具体参考**二进制安全**
  
    ```bash
    127.0.0.1:6379> set key1 a
    OK
    127.0.0.1:6379> get key1
    "a"
    127.0.0.1:6379> STRLEN key1
    (integer) 1
    127.0.0.1:6379> APPEND key1 中    # 汉字在 UTF-8 为3个字节
    (integer) 4
    127.0.0.1:6379> STRLEN key1
    (integer) 4
    127.0.0.1:6379> get key1 # “中”的十六进制表示: \xe4\xb8\xad
    "a\xe4\xb8\xad"
    127.0.0.1:6379>	
    ```

### 1.4 String 字符串形式应用场景

-   传统项目同步 session
-   数据缓存（信息缓存、图片缓存、文件缓存）
-   锁

### 1.5 数值操作

-   `Redis` 存储以`k-v`结构存储，`v`默认是字符串值，没有专用的`整数`和`浮点`类型。如果一个字符串值可以被`Redis内部`解释为`十进制64位有符号的整数、128位有符号的浮点数`时（Redis 内部有一个`RedisObject`的结构体会记录数据类型和编码格式等），则可以执行数值计算操作。

#### 1.5.1 数值常用命令

```bash
# 操作整型数值
# 设置 key 对应的数值增加一个指定的 increment，如果 key 不存在，操作前会设置一个0值
INCRBY key increment

# 设置 key 对应的值执行原子的 +1 操作
INCR key

# 设置 key 对应的数值减去一个指定的 decrement，如果 key 不存在，操作前会设置一个0值
DECRBY key decrement

# 设置 key 对应的值执行原子的 -1 操作
DECR key

# 操作浮点数值，浮点操作与整型操作不一样，没有加减方向，根据 increment 符号确定
# 设置 key 对应的数值增加一个指定的 increment，如果 key 不存在，操作前会设置一个0.0值
INCRBYFLOAT key increment
```

### 1.6 String 数值形式应用场景

-   计数器（统计访问次数）
-   限速器（防止资源滥用）

## 2. Hash

>   进入Redis 客户端使用  help @hash 查看命令
>
>   示例：https://redis.io/commands#hash

**Hash 将一个键和一个散列的字段及其映射的值关联，可以操作字段和值，字段和值可以是 String，也可以是普通的文字、图片、视频等二进制数据**

-   `Redis` 的 `Hash` 是一个 `key-value` 结构。外层的哈希（k-v）用到了`Hashtable`，内存的哈希（value 内的 k-v）则使用了两种数据结构来实现，查看哈希对象的编码命令为：`OBJECT ENCODING <key>`

    -   **ziplist**

        -   当 `Hash` 对象满足两个条件使用 `ziplist`，如果不能满足这两个条件的哈希对象需要使用 `hashtable` 编码，以下两个条件可以通过配置文件修改。
            -   哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节`hash-max-ziplist-value 64`
            -   哈希对象保存的键值对数量小于 512 个`hash-max-ziplist-entries 512`

        ![Ziplist](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/hash-ziplist.jpg)

    -   **hashtable**

        ![Hashtable](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/hash-table.jpg)

### 2.1 Hash 常用命令

```bash
# 设置 Hash 里面一个或多个字段的值
HSET key field value

# 获取 Hash 里面 key 指定字段的值
HGET key field

# 获取 Hash 所有字段
HKEYS key

# 获取 Hash 所有值
HVALS key

# 获取 Hash 所有的 filed 和 value
HGETALL key

# 删除一个或多个 Hash 的 filed
HDEL key field [field ...]

# 判断 filed 是否存在该 key 中
HEXISTS key field

# 将 Hash 中 key 指定的 filed 增加给定的整型数值，前提该 filed 的值可以被 Redis 作为数值解释
HINCRBY key field increment

# 将 Hash 中 key 指定的 filed 增加给定的浮点数值，前提该 filed 的值可以被 Redis 作为数值解释
HINCRBYFLOAT key field increment

# 获取指定 key 的字段数量
HLEN key

# 获取 Hash 里面 key 指定 filed 的字节长度
HSTRLEN key filed

```

### 2.2 Hash 应用场景

-   存储对象缓存

## 3. List

>   进入Redis 客户端使用  help @list 查看命令
>
>   示例：https://redis.io/commands#list

**List 由一个双向链表组成，类似 Java 的 LinkedList**

-   `Redis` 使用 `quicklist` 的双向链表数据结构实现 List。在链表两段插入数据的复杂度为O(1)，在中间操作性能会很差。

    -   `quicklist` 结构

        ![QuickList](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/redis_quicklist_structure.png)

        -   两端各有2个橙黄色的节点，是没有被压缩的。它们的数据指针zl指向真正的 `ziplist`。中间的其它节点是被压缩过的，它们的数据指针zl指向被压缩后的 `ziplist` 结构，即一个 `quicklistLZF` 结构。

        -   左侧头节点上的`ziplist`里有2项数据，右侧尾节点上的`ziplist`里有1项数据，中间其它节点上的 `ziplist` 里都有3项数据（包括压缩的节点内部）。这表示在表的两端执行过多次`push`和`pop`操作后的一个状态。

        -   ```c
            typedef struct quicklist {
                quicklistNode *head;
                quicklistNode *tail;
                unsigned long count;        /* 所有的 ziplists 中的所有节点的总数 */
                unsigned long len;          /* quicklistNodes 的个数 */
                int fill : 16;              /* 每个 quicklistNodes 的 fill factor */
                unsigned int compress : 16; /* 0 表示不进行压缩，否则这个值表示，表的头和尾不压缩的节点的个数  */
            } quicklist;
            
            typedef struct quicklistNode {
                struct quicklistNode *prev;  /* 指向上一个节点 */
                struct quicklistNode *next;  /* 指向下一个节点 */
                unsigned char *zl;           /* 当节点保存的是压缩 ziplist 时，指向 quicklistLZF，否则指向 ziplist */
                unsigned int sz;             /* ziplist 的大小（字节为单位） */
                unsigned int count : 16;     /* ziplist 的项目个数 */
                unsigned int encoding : 2;   /* RAW==1 或者 LZF==2 */
                unsigned int container : 2;  /* NONE==1 或者 ZIPLIST==2 */
                unsigned int recompress : 1; /* 这个节点是否之前是被压缩的 */
                unsigned int attempted_compress : 1; /* 节点太小，不压缩 */
                unsigned int extra : 10; /* 未使用，保留字段 */
            } quicklistNode;
            
            typedef struct quicklistLZF {
                unsigned int sz; /* 压缩后的 ziplist 长度 */
                char compressed[]; /* 压缩后的实际内容 */
            } quicklistLZF;
            ```


### 3.1 常用命令

```bash
# 从队列右边入队一个元素
RPUSH key value [value ...]

# 从队列左边入队一个元素
LPUSH key value [value ...]

# 以队里从左到有发现的第一个元素为轴心，在其指定方向插入一个元素
# 如果一个List 是 a a b c d，在执行 LINSERT k1 after a 1 之后，会变成 a 1 a b c d
LINSERT key BEFORE|AFTER pivot value

# 根据索引设置队列里边一个值
LSET key index value

# 根据 count 移除指定的元素
# count = 0，移除列表所有指定元素
# count > 0，从左到右移除 count 个指定元素
# count < 0，从右到左移除 count 个指定元素
LREM key count value

# 根据指定索引获取一个元素
LINDEX key index

# 根据指定范围获取元素(0,-1 获取所有)
LRANGE key start stop

# 从队列左侧出队一个元素
LPOP key

# 从队列右侧出队一个元素
RPOP key

# 在指定的时间内，一直出于阻塞状态，从左侧出队一个元素
# 如果阻塞状态时，列表元素已经为空，当另外一个执行了 PUSH 命令，则出于阻塞状态的命令会将 PUSH 的元素进行出队操作
BLPOP key [key ...] timeout

# 在指定的时间内，一直出于阻塞状态，从右侧出队一个元素
# 原理同 BLPOP
BRPOP key [key ...] timeout

# 将 源列表的最右端元素（source） 弹出并推入 目标列表的最左端（destination）
RPOPPUSH source destination

# 阻塞状态的 RPOPPUSH，当 source 没有元素时，可以参考 BLPOP
BRPOPLPUSH source destination timeout

```

### 3.2 List 应用场景

-   LREM 可以实现待办事项列表
-   PUSH 相关命令可以实现评论列表等
-   POP 相关的命令可以实现阻塞、非阻塞队列

## 4.Set

>   进入Redis 客户端使用  help @set 查看命令
>
>   示例：https://redis.io/commands#set

**Set 大多数情况是 Hashtable，使用 key 来存储元素，value 设置为 Null**

-   `Redis` 使用 `intset` 编码的整数集合或`hashtable` 编码的集合对象实现`无序` Set。

    -   当 `Set` 内所有元素对象都可以被 `Redis` 解释为整型数值时，且元素数量不超过 `512`  个，使用 `intset` 编码。
    -   当 `Set` 内有元素为 `String ` 类型是，使用 `hashtable` 编码，每个元素对应 `hashtable` 字典的键，值全部设置为 `Null`。

    ![Set](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/set-1.jpg)

### 4.1 Set 常用命令

```bash
# 添加一个或多个元素到集合（set）内
SADD key member [member ...]

# 获取集合内所有元素
SMEMBERS key

# 判断给定的元素是否属于集合成员，是返回1，否返回0
SISMEMBER key member

# 获取集合元素数量
SCARD key

# 移动 源集合（source）的指定元素（member）到目标集合（destination）
SMOVE source destination member

# 移除一个或多个元素
SREM key member [member ...]

# 随机返回一个或 count 个数的元素（不同于 POP 命令，只是返回元素，不会移除）
SRANDMEMBER key [count]

# 随机返回 count 个数元素，并从集合内移除
SPOP key [count]

# 集合交集
SINTER key [key ...]

# 集合并集
SUNION key [key ...]

# 集合差集，按照给定顺序，从左到右计算
# k1(a,b,c)		k2(b,c,d)		SDIFF k1 k2 ==> a
SDIFF key [key ...]

# 计算交集。按照给定的集合，并将结果存储到 destination 中
SINTERSTORE destination key [key ...]

# 计算并集。按照给定的集合，并将结果存储到 destination 中
SUNIONSTORE destination key [key ...]

# 计算差集集。按照给定的集合顺序，从左到右计算，并将结果存储到 destination 中 
SDIFFSTORE destination key [key ...]

```

### 4.2 Set 应用场景

-   SCARD 命令实现`唯一计数器`
-   SADD、SREM、SISMEMBER 命令实现`点赞`、`喜欢`、`Tags`、`投票`、`社交关系（关注）`
-   SRANDMEMBER、SPOP 命令实现`抽奖`
-   SINTER 命令实现社交关系的`共同好友`
-   SUNION 命令实现社交关系的`推荐关注`

## 5. ZSet(Sorted Set)

>   进入Redis 客户端使用  help @sorted_set 查看命令
>
>   示例：https://redis.io/commands#sorted_set

**ZSet 通过 score 进行排序的 Set，默认表头小，表尾大。具体结构看详解**

-   `Redis` 使用 `ziplist` 或者 `skiplist` 的编码实现 `有序的 Sorted set`。

    -   编码转换

        -   有序集合保存的元素数量小于128个，且有序集合保存的所有元素成员的长度都小于64字节，使用 `ziplist`，当元素数量和长度超过规定则使用 `skiplist`。

    -   `ziplist` 编码的压缩列表底层内部，每个集合元素使用两个紧挨一起的压缩列表节点来保存。第一个节点保存元素的`成员（member）`，第二个元素保存元素的`分值（score）`。分值较小的放置在靠近表头的方向，分值较大的放置在靠近表尾的方向。

        ![ziplist](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/ziplist.jpg)

        ![ziplist-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/ziplist-1.jpg)

    -   `skiplist` 编码使用 `zset` 结构作为底层实现。`zset` 结构包含一个`字典(dict)`和一个`跳跃表(zskiplist)`。

        -   `zset` 的 `zsl`按分值从小到大保存了所有集合元素，每个跳跃吧节点保存了一个结合元素。跳跃表的 `object` 属性保存了元素成员， `score` 属性保存了元素分值。`ZRANK、ZRANGE`等基于跳跃表API实现。
        -   `zset` 的 `dict` 为有序集合创建了成员到分值的映射，字典中的每个键值对都保存了一个集合元素。字典的键保存了元素成员，字典的值保存了元素的分值。如果使用 `ZSCORE` 查看成员分值，复杂度为 `O(1)`。
        -   两种数据结构虽然都保存了有序集合元素，但是两种数据结构通过指针共享相同元素的成员和分支，所以不会出现重复的成员或分值。
        -   **采用两种数据结构共同实现的原因：**
            -   单独使用字典：因为字典以无序方式来保存集合元素，当执行范围型操作，比如`ZRANK、ZRANGE`，需要对字典内所有元素排序，时间复杂度为`O(nlogn)`，及额外的`O(N)`内存空间（需要创建数组来保存排序后的元素）。
            -   单独使用跳跃表：当执行 `ZSCORE` 操作是，复杂度为 `O(logN)`.

        ```c
        typedef struct zset {
            zskiplist *zsl; /* 跳跃表 */
            dict *dict; /* 字典 */
        }zset;
        ```

        ![skiplist](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/skiplist.jpg)

### 5.1 ZSet 常用命令

```bash
# 添加一个或多个成员，或更新成员的分数（如果已经存在则更新，不存在添加）
ZADD key [NX|XX] [CH] [INCR] score member [score member ...]

# 对给定成员进行指定的增量（increment）
ZINCRBY key increment member

# 获取成员数量
ZCARD key

# 获取成员分数
ZSCORE key member

# 计算成员之间的成员数量（包含该成员本身，正数后查，负数前查）
ZLEXCOUNT key min max

# 获取分数范围内的成员数量
ZCOUNT key min max

# 按成员位置从低到高排列
ZRANGE key start stop [WITHSCORES]

# 按成员位置从高到底排列
ZREVRANGE key start stop [WITHSCORES]

# 移除指定的成员
ZREM key member [member ...]

# 按成员范围移除
# min 和 max 如果使用 (min (max 则不包含该元素，只移除元素范围内的元素
# min 和 max 如果使用 [min [max 则包含该元素，移除元素和其范围内的元素
ZREMRANGEBYLEX key min max

# 按成员下边索引移除
ZREMRANGEBYRANK key start stop

# 按成员分数移除
ZREMRANGEBYSCORE key min max

# 获取分数范围内的成员（如果使用 WITHSCORES 则会显示分数）
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]

# 计算并集，给定 key 的数量由 numkeys 决定，并将结果保存在 destination 中。
# 如果使用 WEIGHTS 参数，后边的参数因子跟 key 的顺序是对应的。
# ZUNIONSTORE result 2 k1 k2 WEIGHTS 1 2 （k1的成员分数×1，k2的成员分数×2）
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]

# 计算交集，参考上边并集
ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]
```

### 5.2 ZSet 应用场景

-   ZRANGE 命令集合实现排行榜、权重排名

## 6. Bitmap    

**Bitmap(二进制位)属于 String 内的一种数据结构，因其应用场景较多，故单独拿出来。**

>   进入Redis 客户端使用  help @string 查看命令
>
>   示例：https://redis.io/commands#string

### 6.1 Bitmap 图解

-   **使用一个 `bit` 标记一个元素对应的`value`，`Key`即是该元素。**
-   存储一个数组`[2,4,7]`
  -   1.  计算机分配一个`byte`，初始化为8个为`0`的`bit`。
  -   2.  根据数组给定的值，在对应的`offset`将`bit`的值修改为`1`标记该元素。

![Bitmap-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChat6b19a88bb391b20c5e83a940fe8a1366.png)

-   增加一个元素`5`，使数组变为`[2,4,5,7]`
  
	![Bitmap-add-5](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChat024dd7db382d59765da9fcdb8f686a04.png)

-   删除一个元素`4`，使数组变为`[2，7]`

  -   **注意进行与运算时，只有 offset 为 4 的 bit 为 0，其余 bit 都是 1**
  

	![Bitmap-del-4](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChat6dd72906063050e25089960011af57d2.png)


  - 增加一个元素`20`,使数据变为`[2,4,7,20]`。
    
      - 如果现有数组要增加元素，并且元素大于7，则再分配字节，并且`offset`仍然`从右向左`依次标记。例如增加20，则分配三个字节，分别是`buf[0]`、`buf[1]`、`buf[2]`，在`buf[2]`的`offset`对应元素标记为`1`。其中`buf[1]`的所有元素肯定为`0`。

![Bitmap-add](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChat3386d4b0c2c7bfe6ffe00d8059c13d03.png)

### 6.1 Bitmap 常用命令

```bash
# 对 key 所存储的字符串值（bit）按照指定的 offset 进行标记（标记为0或1）
# 实际操作的是每个字节对应的二进制值
SETBIT key offset value

# 获取该 key 对应的 offset 的 bit 值
GETBIT key offset

# 返回二进制 bit 值为 1 的数量
BITCOUNT key [start end]

# 返回指定范围二进制第一个 bit 值的位置
BITPOS key bit [start] [end]

# 对 一个或多个 key 进行位运算，并将结果保存在 destkey 上
# operation 支持以下四种操作任意一种：
#	AND，与（&）  ：对应位都为1，结果为1
#	OR，或（|）   ：对应位有一个为1，结果为1
#	XOR，异或（^） ：对应位不同时，结果为1
#	NOT，非（~）   ：一元操作，取反值(只能对一个 key 操作，不同于上述三种)
BITOP operation destkey key [key ...]
```

-   **演示**

    -   准备工作：

        -   ascii 码表

            ![ascii](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/ascii.jpg)

        -   [二进制计算器](https://cn.calcuworld.com/%e4%ba%8c%e8%bf%9b%e5%88%b6%e8%ae%a1%e7%ae%97%e5%99%a8?iframe=1 ':include :type=iframe width=100% height=400px')（该网站计算的值如果不够8位，需要在高位`左侧`补齐0）
    
-   操作 `Bitmap` 设置一个字符串
  
    ```bash
    # 操作 Bitmap 使其值为十进制的 1
    127.0.0.1:6379> SETBIT k1 7 1  # 0000 0001 
    (integer) 0
    127.0.0.1:6379> get k1
    "\x01"
    
    # 操作 Bitmap 使其值为 a，在 ascii 对应的十进制为 97，用二进制表示为：01100001
    127.0.0.1:6379> SETBIT k2 0 0
    (integer) 0
    127.0.0.1:6379> SETBIT k2 1 1
    (integer) 0
    127.0.0.1:6379> SETBIT k2 2 1
    (integer) 0
    127.0.0.1:6379> SETBIT k2 3 0
    (integer) 0
    127.0.0.1:6379> SETBIT k2 4 0
    (integer) 0
    127.0.0.1:6379> SETBIT k2 5 0
    (integer) 0
    127.0.0.1:6379> SETBIT k2 6 0
    (integer) 0
    127.0.0.1:6379> SETBIT k2 7 1
    (integer) 0
    127.0.0.1:6379> get k2
    "a"
    
    # 操作 Bitmap 的 k2 的值，使 a 更改为 b，
    # 在 ascii 对应的 a 十进制为 97，用二进制表示为：01100001
    # 在 ascii 对应的 b 十进制为 98，用二进制表示为：01100010
    127.0.0.1:6379> SETBIT k2 7 0
    (integer) 1
    127.0.0.1:6379> SETBIT k2 6 1
    (integer) 0
    127.0.0.1:6379> get k2
    "b"
        
    # 统计用户登录登录次数，id 为 1000，1001 的两个用户在 20200515、20200516 时间内登录
    127.0.0.1:6379> SETBIT 1000 20200515 1
    (integer) 0
    127.0.0.1:6379> SETBIT 1001 20200515 1
    (integer) 0
    127.0.0.1:6379> SETBIT 1000 20200516 1
    (integer) 0
    127.0.0.1:6379> BITCOUNT 1000
    (integer) 2
        
    # 统计任意时间范围内用户活跃情况，id 为 1000，1001，1002 的三个用户分别在 20200514、20200514 登录，最后统计两天内活跃人数为 3
    
    127.0.0.1:6379> SETBIT 20200514 1000 1
    (integer) 0
    127.0.0.1:6379> SETBIT 20200514 1001 1
    (integer) 0
    127.0.0.1:6379> SETBIT 20200514 1002 1
    (integer) 0
    127.0.0.1:6379> SETBIT 20200515 1000 1
    (integer) 0
    127.0.0.1:6379> SETBIT 20200515 1001 1
    (integer) 0
    127.0.0.1:6379> BITOP or result 20200514 20200515
    (integer) 126
    127.0.0.1:6379> BITCOUNT result
    (integer) 3
    127.0.0.1:6379>
    ```
    

### 6.2 Bitmap 使用场景（用户 id 只能是数字类型）

-   统计任意时间内用户登录次数
    -   SETBIT `user:id` time 1
    -   BITCOUNT `user:id`
-   统计用户登录在某一时间段内活跃情况
    -   SETBIT `time` user-id 1
    -   BIETOP OR  result time1 time2
    -   BITCOUNT result

### 6.3 SETBIT 和 GETBIT 的底层原理

-   `Redis` 的 `SETBIT` 命令存储`Bitmap`时，采用的`逆序存储`，该逆序顺序对于用户来讲是无感知的。主要目的是`扩展字节数据时，不再需要移动数据`。

-   以下内容摘抄自**黄健宏的《Redis 设计与实现》** 章节。

-   `Redis`使用字符串对象保存位数组。因为字符串对象使用的`SDS`数据结构是二进制安全的。

    -   redisObject.type的值为REDIS_STRING，表示这是一个字符串对象。

    -   sdshdr.len的值为1，表示这个SDS保存了一个一字节长的位数组。

    -   buf数组中的buf[0]字节保存了一字节长的位数组。

    -   buf数组中的buf[1]字节保存了SDS程序自动追加到值的末尾的空字符'\0'。

        ![SDS-Structure](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-structure.jpg)

    -   存储一个`字节`的位数组：`0100 1101`，`Redis`在保存数组顺序时，与我们书写顺序时完全相反的。也就是`逆序存储`，在数组中的表示为：`1011 0010`。**注意：数组索引顺序依然是从左到右，不是逆序存储的时候采用了逆序索引，这点在后续会明确解释。**

        ![SDS-Structure](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-structure-1.jpg)

    -   存储多个`字节`的位数组：`1111 0000 1100 0011 1010 0101`，在 `buf数组中`表示为：`1010 0101 1100 0011 0000 1111`。

        ![SDS-Structure](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-structure-2.jpg)

    -   `GETBIT <bitarray> <offset>` 命令实现，复杂度为`O(1)`

        -   1. **byte = offset ÷ 8 并向下取整（byte 值是图22-3的 buf[0]、buf[1]、buf[2]  ）**

            -   byte 值记录了`offset` 指定的二进制位保存在位数组的哪个字节。

        -   2. **bit = ( offset mod 8 ) + 1**
        
            -   bit 值记录了`offset`指定的二进制位中字节的第几个位置（**不是索引，是位置**）。
        
        -   3.  根据 `byte` 和 `bit` 值，在位数组 `bitarray` 中定位 `offset` 指定的二进制位，返回该位上的值。
        
        -   **示例：**`GETBIT <bitarray> 3`
        
            ![GETBIT-3](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/GETBIT-3.jpg)
        
        -   **示例：**`GETBIT <bitarray> 10`
        
            ![GETBIT-10](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/GETBIT-10.jpg)
        
    -   `SETBIT <bitarray> <offset> <value>` 命令实现，复杂度为`O(1)`

        -   1. **len = ( offset ÷ 8 ) + 1**

            -   len 值记录了保存`offset`指定的二进制位需要多少字节（计算 bug[] ）

        -   2. 检查 `bitarray` 键保存的位数组**(sdshdr)**的长度是否小于 len。

            -   2.1 如果小于 len，则需要扩展字节。`sdshdr` 空间预分配策略会额外多分配 `len` 个字节的未使用空间，再加上为保存空字符而额外分配的1字节，及扩展后的字节为：**( len × 2 ) + 1**

        -   3.  **byte = offset ÷ 8 并向下取整**

            -   byte 值记录了`offset` 指定的二进制位保存在位数组的哪个字节。

        -   4.  **bit = ( offset mod 8 ) + 1**

            -   bit 值记录了`offset`指定的二进制位中字节的第几个位置（**不是索引，是位置**）。

        -   5.  根据 **byte** 和 **bit** 值，在 `bitarray` 键保存的位数组中定位`offset`指定的二进制位。

            -   首先将指定的二进制位上当前值保存在 `oldvalue` 变量。
            -   然后将新值 `value` 设置为二进制位的值。

        -   6.  返回 `oldvalue` 变量值。

        -   **示例：**`SETBIT <bitarray> 1 1` 无需扩展字节

            ![SETBIT-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIT-1.jpg)

            ![SETBIE-1-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIT-1-1.jpg)

        -   **示例：**`SETBIT <bitarray> 12 1`需扩展字节

            ![SETBIT-12](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIT-12.jpg)
            
            ![SETBIT-12-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIE-12-1.jpg)
          
        -   **SETBIT 如果采用正常书写顺序保存，在每次扩展buf数组之后，程序都需要将位数组已有的位进行移动，然后才能执行写入操作，这比SETBIT命令目前的实现方式要复杂，并且移位带来的CPU时间消耗也会影响命令的执行速度。对位数组0100 1101执行命令SETBIT ＜bitarray＞ 12 1，将值改为0001 0000 0100 1101的整个过程。如下图。**

            ![SETBIT-ORDER](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIE-12-3.jpg)
          
        -   **关于`逆序存储` 的疑问及解释，如果根据上文的逆序存储方式进行验证，会出现以下几个疑问。最后的 Redis 源码解释了该问题，通过 bit = 7 - ( bitoffset & 0x7 ) 计算，实际上的 setbitCommand 操作将 0 1 2 3 4 5 6 7 的操作倒转为了 7 6 5 4 3 2 1 0。对于用户来讲，该操作是无感知的，所以当验证逆序存储是，就会出现了下边几个疑问。**

            ```c
            /* GET current values*/
            // 将指针定位到要设置的为所在的字节上
            byteva1 = ((uint8_t*)o->ptr)[byte];
            // 定位到要设置的位上面
            // 此处是逆序存储的关键步骤，将 0 1 2 3 4 5 6 7 的操作倒转为了 7 6 5 4 3 2 1 0
            bit = 7 - (bitoffset & 0x7)
            // 记录位现在的值
            bitva1 = byteva1 & (1 << bit);
            // 更新字节中的位，设置它的值为 on 参数的值
            byteva1 &= ~(1 << bit);
            byteva1 |= ((on & 0x1) << bit);
            ((uint8_t*)o->prt)[byte] = byteva1 
            ```

            -   **Question-1**

              ![Question-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-1.jpg)

            -   **Question-2**

              ![Question-2](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-2.jpg)

            -   **Question-3**

              ![Question-3](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/%20bitmap-3.png)



## 7. HyperLogLog

>   进入Redis 客户端使用     help @hyperloglog
>
>   示例：https://redis.io/commands#hyperloglog

**HyperLogLog	计算一个集合内近似元素个数的概率算法，Redis 做了去重处理**

**HyperLogLog 通过记录二进制出现的第一个 1 来实现基数计数**

**HyperLogLog 内部结构的本质是 16384 个桶（一个桶是 6bit）的 Bitmap**

`HyperLogLog` 是一个专门为了计算集合的基数而创建的概率算法，对于一个给定的集合，`HyperLogLog` 可以计算出这个集合的近似基数：近似基数并非集合的实际基数，它可能会比实际的基数小一点或者大一点，但是估算基数和实际基数之间的误差会处于一个合理的范围之内，因此那些不需要知道实际基数或者因为条件限制而无法计算出实际基数的程序就可以把这个近似基数当作集合的基数来使用。

`HyperLogLog` 的优点在于它计算近似基数所需的内存并不会因为集合的大小而改变，无论集合包含的元素有多少个，`HyperLogLog` 进行计算所需的内存总是固定的，并且是非常少的。
具体到实现上，`Redis` 的每个 `HyperLogLog` 只需要使用 `12KB` 内存空间，就可以对接近：**2<sup>64</sup>** 个元素进行计数，而算法的标准误差仅为 `0.81%`，因此它计算出的近似基数是相当可信的。

### 7.1 HyperLogLog 算法

-   **基数计数基本概念**
  -   在数学上，**基数或势**，即集合中包含的元素的 `个数`。有限集合的基数，其意义与日常用语中的 `基数` 相同，例如 `{a,b,c}` 的基数是 `3`。
    -   **基数计数(Cardinality Counting)** 通常用来统计一个集合中不重复的元素个数，例如统计某个网站的UV，或者用户搜索网站的关键词数量。数据分析、网络监控及数据库优化等领域都会涉及到基数计数的需求。 要实现基数计数，最简单的做法是记录集合中所有不重复的元素集合 **S<sub>u</sub>**，当新来一个元素 **x<sub>i</sub>**，若 **S<sub>u</sub>** 中不包含元素 **x<sub>i</sub>**，则将 **x<sub>i</sub>** 加入 **S<sub>u</sub>**，否则不加入，计数值就是 **S<sub>u</sub>** 的元素数量。这种做法存在两个问题：
        -   1.  当统计的数据量变大时，相应的存储内存也会线性增长
        -   2.  当集合 **S<sub>u</sub>** 变大，判断其是否包含新加入元素 **x<sub>i</sub>** 的成本变大
    -   **基数基数方法**
        -   **B-Tree** --- `占用空间较大，空间随数据量线性增长`
            -   `B-Tree` 最大的优势是`插入`和`查找`效率很高。
            -   如果用 `B-Tree` 存储要统计的数据，可以快速判断新来的数据是否已经存在，并快速将元素插入 `B-Tree` 。要计算基数值，只需要计算 `B-Tree` 的节点个数。 将 `B-Tree` 结构维护到内存中，可以快速统计和计算，但依然存在问题， `B-Tree` 结构只是加快了查找和插入效率，并没有节省存储内存。
        -   **Bitmap** --- `占用空间大，空间随数据量线性增长`
            -   `Bitmap` 最大的优势是可以轻松合并多个统计结果，只需要对多个结果求异或。
            -   `Bitmap` 可以理解为通过一个 `bit数组` 来存储特定数据的一种数据结构，每一个`bit位` 都能独立包含信息，`bit` 是数据的最小存储单位，因此能大量节省空间，但随着数据量增长，空间也会线性增加。
        -   **概率算法** --- `占用空间最小，结果有一定偏差`
          
            -   大数据场景没有更好的准确计算基数的高效算法。在不追求绝对准确的前提下，使用`概率算法`是一个较好的解决方案。
            -   `概率算法`不直接存储数据集合本身，通过一定的概率统计方法预估基数值，这种方法可以大大节省内存，同时保证误差控制在一定范围内。
            -   目前用于基数计数的`概率算法`包括:
                -   **Linear Counting(LC)**：早期的基数估计算法，`LC` 在空间复杂度方面并不算优秀，实际上 `LC` 的空间复杂度与上文中 `Bitmap` 方法是一样的（但是有个常数项级别的降低），都是 **O(N<sub>max</sub>)**
                -   **LogLog Counting(LLC)**：`LLC` 相比于 `LC` 更加节省内存，空间复杂度只有 **O(log<sub>2</sub>(log<sub>2</sub>(N<sub>max</sub>)))**
                -   **HyperLogLog Counting(HLL)**：`HHL` 是基于 `LLC` 的优化和改进，在同样空间复杂度情况下，能够比 `LLC` 的基数估计误差更小。
            -   **HLL 演示网站**
            -   [HyperLogLog Demo](http://content.research.neustar.biz/blog/hll.html) 讲解了 `HLL` 具体的计算方法及计算步骤
            -   **HLL 占用的内存空间**
            -   假如使用 `Bitmap` 存储一亿个统计数据大概需要 `12M` 内存。而在 `HLL` 中，只需要不到 `1K` 内存就能做到。
                -   `Redis` 中实现的 `HyperLogLog`，只需要 `12K` 内存，在标准误差 `0.81%` 的前提下，能够统计 **2<sup>64</sup>** 个数据。
            -   **HLL 基本原理**
                -   `HLL` 实际存储的是一个长度为 `m` 的大数组 `S`，将待统计的数据集合划分成 `m`组
                -   每组根据算法记录一个统计值存入数组中。数组的大小 `m` 由算法实现方自己确定，`Redis` 中这个数组的大小是 **16834**，`m` 越大，基数统计的误差越小，但需要的内存空间也越大。
                -   **伯努利实验（分布离散型概率）**
                    -   **伯努利试验**（Bernoulli trial）(或译为**白努利试验**)是只有两种可能结果（**成功**或**失败**）的单次[随机试验](https://zh.wikipedia.org/wiki/隨機試驗)，即对于一个[随机变量](https://zh.wikipedia.org/wiki/隨機變量)X而言，
                    -   举一个我们最熟悉的抛硬币例子，出现正反面的概率都是 `1/2`，一直抛硬币直到出现正面，记录下投掷次数 `k`，将这种抛硬币多次直到出现正面的过程记为一次**伯努利过程**。
                    -   假设每次伯努利试验所经历了的抛掷次数为 `k`。第一次伯努利试验，直到抛出正面次数设为 `k1`，以此类推，第 `n` 次对应的是 `k_n`，那么必然有一个最大的抛掷次数 `k_max`。则得出以下 `2` 个结论：
                        -   `n` 次伯努利过程的投掷次数都满足 `k <= k_max`（几何分布）
                        -   `n` 次伯努利过程，至少有一次的投掷次数满足 `k = k_max`（二项分布）
                    -   实验结果
                        -   第 1 次试验: 抛了 3 次才出现正面，此时 k=3，n=1
                            第 2 次试验: 抛了 2 次才出现正面，此时 k=2，n=2
                            第 3 次试验: 抛了 6 次才出现正面，此时 k=6，n=3
                            第 n 次试验: 抛了 12 次才出现正面，估算 n=2<sup>12</sup>
                    -   实验结论：
                        -   如果认为硬币正面为 **1**，第一次抛到硬币正面的次数记为 `k`，`n` 次实验出现的最大值为 `k_max`，则出现数据集合的基数公式为：**$\widehat{n}$ = 2<sup>k_max</sup>** `此处 n 的显示应该是 n 带帽一个 ^ 符号，Github 的 MD 语法不支持，意思为 n 无穷大`
                        -   只有 `n` 足够大时，误差率才会减小 
                        -   `Redis` 的 `HyperLogLog` 使用 `Bitmap` 记录第一次 `1` 的位置来计算基数
  
-   **HyperLogLog 算法** 

    -   **分桶平均思想**
        -   `HLL` 的基本思想是利用集合中数字的比特串第一个`1`出现位置的最大值来预估整体基数，但是这种预估方法存在较大误差，为了改善误差情况，`HLL` 中引入分桶平均的概念。
        -   同样举抛硬币的例子，如果只有一组抛硬币实验，运气较好，第一次实验过程就抛了10次才第一次抛到正面，显然根据公式推导得到的实验次数的估计误差较大；如果100个组同时进行抛硬币实验，同时运气这么好的概率就很低了，每组分别进行多次抛硬币实验，并上报各自实验过程中抛到正面的抛掷次数的最大值，就能根据100组的平均值预估整体的实验次数了。
        -   分桶平均的基本原理是将统计数据划分为 `m` 个桶，每个桶分别统计各自的 `k_max` 并能得到各自的基数预估值 **$\widehat{n}$** ，最终对这些 **$\widehat{n}$** 求平均得到整体的基数估计值。`LLC` 中使用几何平均数预估整体的基数值，但是当统计数据量较小时误差较大；`HLL` 在 `LLC` 基础上做了改进，采用调和平均数，调和平均数的优点是可以过滤掉不健康的统计值。
        -   **调和平均数公式**（两者等同）
            -   ![Harmonic](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/harmonic.svg)
            -   ![Harmonic-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/harmonic-1.svg)
        -   **Redis 使用分桶数组原因**
            -   分桶数组是为了消减因偶然性带来的误差，提高预估的准确性。
    -   **偏差修正**
      -   使用**分桶平均**后，虽然误差可以减少很多，但是无法做到无偏估计。[Loglog Counting of Large Cardinalities](http://algo.inria.fr/flajolet/Publications/DuFl03-LNCS.pdf) 的作者提供了一种分阶段修正的算法，感兴趣的可以深入了解。
    
### 7.2 Redis 的 HyperLogLog 数据结构

![HyperLogLog-Bucket](https://gitee.com/bonismo/notebook-img/raw/f8fa4987d5e09afdf5a3507ba6a29f380333fcfa/img/redis/HyperLogLog.jpg)

-   `Redis` 的 `HyperLogLog` 占用 **12k** 内存的原因
    -   `Redis` 设置了**16384（2<sup>14</sup>）** 个桶。请留意 14 这个数字，接下来会具体解释。
        -   每个桶有 **6 个bit**，每个桶可以表达的最大二进制：**111 111**，对应的十进制：**2<sup>5</sup> + 2<sup>4</sup> + 2<sup>3</sup> + 2<sup>2</sup> + 2<sup>1</sup> = 63**。
        -   `HyperLogLog` 占用的内存为：**16384 * 6 / 8 / 1024 = 12k  等价于 2<sup>14</sup> * 6 / 8 / 1024 = 12 k**
        -   `Redis` 的 `HyperLogLog` 内部维护了 **16384 个桶（bucket）**来记录各自桶内元素数量。每个桶内有 **6 个bit**，这 **6个bit** 自然无法容纳桶中元素，在 `Redis` 中记录的是**value 被哈希为 64 位 bit 后 50 位 bit 从右到左第一次出现 1 的索引位置**。
-   **16384 的桶是否和 16384 个 Slot 有关系呢？**
    -   我也不清楚。但是`Redis` 的作者 **antirez** 解释了采用 **2<sup>14</sup>** 的原因，可以简单的理解为：当使用 `Redis` 集群时，`Redis` 节点发送心跳包时，需要使用 `Bitmap` 压缩到消息头。**16384 / 8(bit) / 1024(k) = 2k**，那么这 **2k** 数据存储的什么呢？消息头有一个 `myslots` 的 `char` 数组（**本质是bitmap**），每一个 `bit` 代表一个槽，如果该位为 **1**，则代表这个槽是映射到该节点的。
    -   如果采用 **2<sup>16</sup> = 65536**，则消息头占用的空间为：**65536 / 8(bit) / 1024(k) = 8k**
    -   [antirez 的回答](https://github.com/antirez/redis/issues/2576)
        -   1.  Normal heartbeat packets carry the full configuration of a node, that can be replaced in an idempotent way with the old in order to update an old config. This means they contain the slots configuration for a node, in raw form, that uses 2k of space with16k slots, but would use a prohibitive 8k of space using 65k slots.
        -   2.  the same time it is unlikely that Redis Cluster would scale to more than 1000 mater nodes because of other design tradeoffs.
        -   So 16k was in the right range to ensure enough slots per master with a max of 1000 maters, but a small enough number to propagate the slot configuration as a raw bitmap easily. Note that in small clusters the bitmap would be hard to compress because when N is small the bitmap would have slots/N bits set that is a large percentage of bits set.

### 7.3 Redis 的 HyperLogLog 原理（从右到左查找第一个出现的 1 ）

-   [Redis HyperLogLog 源码](https://github.com/antirez/redis/blob/unstable/src/hyperloglog.c)
-   `Redis` 使用了 **CRC16 算法** 对 `key` 进行哈希然后**对 16384 求余**，定位所在的 **槽（Slot）**，此处 `HyperLogLog` 存储过程使用的也是该算法。
-   **存储过程**
     -   **1. 计算桶坐标**
        -   `value` 在存入时，会被 `hash` 成 **64 个 bit** 。前 **14** 位 `bit` 用来计算 `value` 的 `bit` 数组（**从右到左**）第一个 **1** 出现的下标位置，然后利用下标的索引数值用来计算存入哪个桶中。
            -   例如：设第一个 **1** 出现位置的数值为 `index` 。当 `index=2` 时，`hash` 后的 `bit 数组`简单表示为: `...000010 [01 0000 0000 0000...]`，**从右到左看**，则前 **14** 位的二进制可以换算成**十进制的 2**，那么 `index` 会被转化后放入编号为 **2** 的桶。
                -   选择 `hash` 后的 **前 14 位** 来标记桶号，因为 **2<sup>14</sup> = 16384** ，最大值正好可以将桶利用完，不浪费空间。
     -   **2. index 转化，并在桶内存入值**
        -   `value` 被 `hash` 后的 **64 位减去 14 位后，剩余 50 位从右到左** ，第一个出现 **1** 的位置再换算成**二进制**。然后将二进制设置到桶中。
        -   每个桶有 **6 个bit**，每个桶可以表达的最大二进制：**111 111**，对应的十进制：**2<sup>5</sup> + 2<sup>4</sup> + 2<sup>3</sup> + 2<sup>2</sup> + 2<sup>1</sup> = 63**。所以极端情况第 **50** 位出现第一个 **1** 也不会越界。
        -   例如：极端情况下，第一个出现 **1** 的位置在第 **50** 位，即 `index = 50`，**转换为二进制为**：`110010`

     -   **3. 原桶内已存储值的情况（保留 index 值最大）**
        -   如果前 14 位一样，则比较 `新 index 值` 是否大于 `原来的 index 值`。只有大于已有值时进行设置。
-   **统计基数**
  -   最终地，一个 `key` 所对应的 **16384** 个桶都设置了很多的 `value` 了，每个桶有一个`k_max`。此时调用 `PFCOUNT` 时，按照前面介绍的估算方式，可以计算出 `key` 的设置了多少次 `value`，也就是统计值。
-   **存储上限**
  -   `value` 被转为 **64** 位的 `bit 数组` ，最终被按照上面的做法记录到每个桶中去。**64 位转为十进制**就是：**2<sup>64</sup>**，`HyperLogLog` 仅用了：`16384 * 6 /8 / 1024 K` 存储空间就能统计多达 **2<sup>64</sup>** 个数。
-   **偏差修正**

      -   **伪代码，浮点数就是修正因子数，可以看出每个桶的因子数不一样**

```c
// m 为桶数
switch (p) {
	case 4:
		constant = 0.673 * m * m;
	case 5:
		constant = 0.697 * m * m;
	case 6:
		constant = 0.709 * m * m;
    default:
		constant = (0.7213 / (1 + 1.079 / m)) * m * m;
	}
```

### 7.4 HyperLogLog 存储结构

-  **稀疏存储结构与密集存储结构**
    -  **稀疏存储结构**
        -  `HyperLogLog` 使用 `ZERO`、`XZERO` 和 `VAL` 来表示稀疏存储。
            -  **字节长度**
                -  其中 `ZERO` 和 `VAL` 长度是 **1** 字节，`XZERO` 长度是 **2** 字节。
            -  **稀疏格式**
                -  `ZERO` 格式：**00 xxx xxx**
                    -  **00** 表示计数值连续为 **0** 
                    -  **xxx xxx（表示为长度 len）** 换算成十进制的值 N 再加 1 ，就是**连续 N + 1 个 bit 位上的值标记为 0**，因为 `111 111` 的十进制为：**63**，所以 `ZERO` 格式**最多只能表示连续 64 个 0**。
                    -  例如：**00 010 101**，`010 101` 换算成十进制是 21，表示为连续 22 个 bit 位上的值为 0
                -   `XZERO` 格式：**01 xxx xxx yyyy yyyy**
                    -   **xxx xxx yyyy yyyy（表示为长度 len）** 换算成十进制表示连续了多少个 0 。最多表示 **16384（111 111 1111 1111 = 2<sup>14</sup>）**。
                    -   例如：**01 111 111 10110101**，`111 111 10110101` 换算成十进制是 16309，表示连续 16309 个 0。
                -  `VAL`格式：**1 vvv vvv xx**
                    -  **vv vvv（换算成十进制为 V，V + 1 表示桶内存储的值）** 
                    -  **xx （换算成十进制为 N，N + 1 表示连续 N + 1 个桶）** 
                    -  例如：**1 00 110 01**，`00 110`换算成十进制是 **6**，`01`换算成十进制是 **1**，则表示为连续 **2 个桶存储的计数值都是 7.**
    -  **密集存储结构**
        -  密集存储比较简单，就是**连续 16384 个6bit 的串成的 Bitmap**。由于每个桶是 `6bit`，而一个字节是 `8bit` ，因此对桶的定位要麻烦一些，需要使用下边的算法。**字节位序是左低右高，日常书写位序是左高右低**，所以需要倒置。
        -  **定位：桶编号为 regnum，桶内计数值从起始位置到现在的偏移为 offset_bytes，一个字节内 bit 的起始位置偏移为 offset_bits**
            -  **offset_bytes = ( regnum * 6 ) / 8** 	求商，计算字节偏移
            -  **offset_bits = ( regnum * 6 ) % 8**      求余，计算桶内从第几位开始计数的。offset_bits 大于 2 则跨越了字节边界，需要拼接两个字节的位片段。如果小于 2 ，这 `6bit` 在一个字节内部。
            -  例如：**桶编号为 2** ，则 ( 2 * 6 ) / 8 = 1，表示**第 2 个字节**，( 2 * 6 ) % 8 = 4，表示第 2 个字节**第 5 个位**是计数值。
-  **稀疏结构转换为密集存储结构条件：**
    -  1. 任意一个计数值从 **32** 变成 **33**，因为 `VAL` 指令已经无法容纳，它能表示的计数值最大为 **32**
    -  2. 稀疏存储占用的总字节数超过 **3000** 字节，这个阈值可以通过 `hll_sparse_max_bytes` 参数进行调整。
-  此处内容较为抽象，如有描述不清楚，请根据参考文章详细阅读或者自行查阅相关资料。

### 7.5 HyperLogLog 常用命令

```bash
# 添加一个或多个元素（自动去重）
# 如果添加一个元素时，该元素已经计过数，则返回0，如果没有计数，则返回1
PFADD key element [element ...]

# 查看一个或多个 key 内包含的元素近似基数
PFCOUNT key [key ...]

# 合并多个 HyperLogLog 为一个，并保存到 destkey 中。（去重多个 HyperLogLog 内元素）
PFMERGE destkey sourcekey [sourcekey ...]
```

### 7.4 HyperLogLog 应用场景

-   大数据统计月活、日活
-   数据量不要求精确，允许少许误差的都可以，比 `Bitmap` 更节省内存

## 8. Geohash

>   进入Redis 客户端使用     help @geo
>
>   示例：https://redis.io/commands#geo

### 8.1 空间索引

- **维基百科：空间索引**（空间数据库（存储与空间中的对象有关的信息的数据库））使用**空间索引**来优化[空间查询](https://en.wikipedia.org/wiki/Spatial_query)。
- 传统的索引类型不能有效地处理空间查询，**例如：两个点相差多远，或者给定一个点的坐标，如何查找附近范围内或者最近的点。**常见的空间索引方法包括以下：
  - [Geohash](https://en.wikipedia.org/wiki/Geohash)
  - [HHCode](https://en.wikipedia.org/wiki/HHCode)
  - [Grid (spatial index)](https://en.wikipedia.org/wiki/Grid_(spatial_index))
  - [Z-order (curve)](https://en.wikipedia.org/wiki/Z-order_(curve))
  - [Quadtree](https://en.wikipedia.org/wiki/Quadtree)
  - [Octree](https://en.wikipedia.org/wiki/Octree)
  - [UB-tree](https://en.wikipedia.org/wiki/UB-tree)
  - [R-tree](https://en.wikipedia.org/wiki/R-tree): Typically the preferred method for indexing spatial data.[*[citation needed](https://en.wikipedia.org/wiki/Wikipedia:Citation_needed)*] Objects (shapes, lines and points) are grouped using the [minimum bounding rectangle](https://en.wikipedia.org/wiki/Minimum_bounding_rectangle) (MBR). Objects are added to an MBR within the index that will lead to the smallest increase in its size.
  - [R+ tree](https://en.wikipedia.org/wiki/R%2B_tree)
  - [R* tree](https://en.wikipedia.org/wiki/R*_tree)
  - [Hilbert R-tree](https://en.wikipedia.org/wiki/Hilbert_R-tree)
  - [X-tree](https://en.wikipedia.org/wiki/X-tree)
  - [kd-tree](https://en.wikipedia.org/wiki/Kd-tree)
  - [m-tree](https://en.wikipedia.org/wiki/M-tree) – an m-tree index can be used for the efficient resolution of similarity queries on complex objects as compared using an arbitrary metric.
  - [Binary space partitioning](https://en.wikipedia.org/wiki/Binary_space_partitioning) (BSP-Tree): Subdividing space by hyperplanes.

### 8.2 Geohash 算法

- [Geohash](https://en.wikipedia.org/wiki/Geohash?spm=a2c6h.12873639.0.0.bd7e46ceCqo932) 是一种地理编码，由 [Gustavo Niemeyer](https://en.wikipedia.org/w/index.php?title=Gustavo_Niemeyer&action=edit&redlink=1) 发明的。它是一种分级的数据结构，把空间划分为网格。Geohash 属于空间填充曲线中的 Z 阶曲线（[Z-order curve](https://en.wikipedia.org/wiki/Z-order_curve)）的实际应用。

- **Z 阶曲线：** 简单的可以理解为使用 **Z 曲线** 使 `Z 和 Z` 的首尾相连，来**填充多维空间**，从而是多维降到一维，最终使其可以用一个数组来表示。

  - 二位空间的 **Z阶曲线**，只需要把每个 Z 首尾相连即可。

  ![Z](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/z.png)

  - **Z 阶曲线**同样可以扩展到**三维空间**。只要 Z 形状足够小并且足够密，也能填满整个三维空间。

  ![Z](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/z-1.png)

- **Geohash** 算法的理论基础就是基于 **Z 阶曲线** 的生成原理。

  - 地球纬度区间是`[-90,90]`,经度区间是`[-180,180]`。 将它展开想象成一个矩形，如下图所示：

    ![Z-3](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/z-3.png)

  - 如果将地球的表面转换成二维空间的平面。我们先将平面切割成四个正方形，然后用简单的 `0 和 1` 编码来标识这个四个正方形，最后按照编码的大小将四个正方形连接起来，这样整个平面就转换成了一条 **Z 阶曲线** ，变成了一维。我们递归对每个正方形做同样的操作，递归的层次越深，整个平面就逐渐被 **Z 阶曲线** 填充。我们的点也会落在每个小正方形里面，小正方形越小，精度就越高。

    - **经纬度二进制组合方式：偶数下标放经度，奇数下标放纬度**
      - **当将空间划分为四块时候，经度左半区间为 0，右半区间为 1，纬度下半区间为 0，上半区间为 1，于是经纬度二进制编码交叉组合，得到编码的顺序分别是左下角 00，左上角 01，右下脚 10，右上角 11。**
    - **如下图所示：**

    ![Z-2](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/z-2.png)

  - 以上3个图简单的展示了 **Z 阶曲线** 如何**将一个三维空间最终降维到一维空间**。

- **Geohash 提供任意精度的分段级别。一般分级从 1-12 级。**

  - | 字符串长度 |      | Cell 宽度 |      | Cell 高度 |
    | :--------: | :--: | :-------: | :--: | :-------: |
    |     1      |  ≤   | 5,000 km  |  ×   | 5,000 km  |
    |     2      |  ≤   | 1,250 km  |  ×   |  625 km   |
    |     3      |  ≤   |  156 km   |  ×   |  156 km   |
    |     4      |  ≤   |  39.1 km  |  ×   |  19.5 km  |
    |     5      |  ≤   |  4.89 km  |  ×   |  4.89 km  |
    |     6      |  ≤   |  1.22 km  |  ×   |  0.61 km  |
    |     7      |  ≤   |   153 m   |  ×   |   153 m   |
    |     8      |  ≤   |  38.2 m   |  ×   |  19.1 m   |
    |     9      |  ≤   |  4.77 m   |  ×   |  4.77 m   |
    |     10     |  ≤   |  1.19 m   |  ×   |  0.596 m  |
    |     11     |  ≤   |  139 mm   |  ×   |  149 mm   |
    |     12     |  ≤   |  37.2 mm  |  ×   |  18.6 mm  |

    **字符串长度：** `经度` 和 `纬度` 的**二进制组合（偶数下标放经度，奇数下标放纬度）**后，每 **5** 位二进制转换位十进制后，采用 **Geohash** 编码转换而来。

    - 当一个经纬度的坐标位于一个地图的网格中，假如该字符串长度是5，则临近格子的字符串前缀（4个字符）完全一致。**如下图：银科大厦的 Hash 字符串位 wx4eqw，周围相邻的前 5 位一样。**

      ![6-5](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/yinke.png)

    - 可以在该网站进行测试。[GeoHash-Map](http://geohash.gofreerange.com/)

  - **精度问题：** 仔细观察精度分级会发现相邻级别相差比较大，如果需求是 50 km，则选字符串长度 3 太大，选字符串 4 又太小。后边会简单介绍 **Google S2** 算法及精度分级。

- **Geohash 常用算法：Base 32 和 Base 36** 

  - **Base 32（2<sup>5</sup> = 32，所以二进制使用 5 位，不足 5 位用 0 补齐 ）**

  - |   Decimal   |   0    |   1    |   2    |   3    |   4    |   5    |   6    |   7    |   8    |   9    |   10   |   11   |   12   |   13   |   14   |   15   |
    | :---------: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: |
    | **Base 32** |   0    |   1    |   2    |   3    |   4    |   5    |   6    |   7    |   8    |   9    |   b    |   c    |   d    |   e    |   f    |   g    |
    | **Decimal** | **16** | **17** | **18** | **19** | **20** | **21** | **22** | **23** | **24** | **25** | **26** | **27** | **28** | **29** | **30** | **31** |
    | **Base 32** |   h    |   j    |   k    |   m    |   n    |   p    |   q    |   r    |   s    |   t    |   u    |   v    |   w    |   x    |   y    |   z    |

    **Base 36，对大小写敏感（没有查到相关资料， [Wiki -Base 36](https://en.wikipedia.org/wiki/Geohash-36)，但是业界确实有使用该算法的，目前只查到 Base 36 不使用 Z 阶曲线或类型学关系，[Ruby 实现的 Geohash36](https://github.com/clothesnetwork/geohash36)）**
    
    |   Decimal   |   0    |   1    |   2    |   3    |   4    |   5    |   6    |   7    |   8    |   9    |   10   |   11   |   12   |   13   |   14   |   15   |   16   |   17   |
    | :---------: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: |
    | **Base 36** |   2    |   3    |   4    |   5    |   6    |   7    |   8    |   9    |   b    |   B    |   C    |   d    |   D    |   F    |   g    |   G    |   h    |   H    |
    | **Decimal** | **18** | **19** | **20** | **21** | **22** | **23** | **24** | **25** | **26** | **27** | **28** | **29** | **30** | **31** | **32** | **33** | **34** | **35** |
    | **Base 36** |   j    |   J    |   K    |   I    |   L    |   M    |   n    |   N    |   P    |   q    |   Q    |   r    |   R    |   t    |   T    |   V    |   W    |   X    |

- **使用 Base 32 对坐标进行区间划分，采用 6 个字符的 GeoHash，则经纬度解析分别需要 15 次（因为每 5 位二进制转换一次十进制，十进制再对应 Base 32 取值），值在左区间取 0，在右区间取 1**

- **以上图银科大厦的坐标 [ Lat:39.981498991524006, Lng:116.3063484933343 ]，对应的字符值为 wx4eqw，可以进行如下解析：**

  - 如果要实际测试，一定要将地图放大为最大，拿到精确的经纬值，否则很有可能解析不出正确的结论。

  - **纬度解析 Lat：39.981498991524006**
  
    |     左区间      |       中值       |    右区间    | 二进制结果 |
    | :-------------: | :--------------: | :----------: | :--------: |
    |       -90       |        0         |      90      |     1      |
    |        0        |        45        |      90      |     0      |
    |        0        |       22.5       |      45      |     1      |
    |      22.5       |      33.75       |      45      |     1      |
    |      33.75      |      39.375      |      45      |     1      |
    |     39.375      |     42.1875      |      45      |     0      |
    |     39.375      |     40.78125     |   42.1875    |     0      |
    |     39.375      |    40.078125     |   40.78125   |     0      |
    |     39.375      |    39.7265625    |  40.078125   |     1      |
    |   39.7265625    |   39.90234375    |  40.078125   |     1      |
    |   39.90234375   |   39.990234375   |  40.078125   |     0      |
    |   39.90234375   |  39.9462890625   | 39.990234375 |     1      |
    |  39.9462890625  |  39.96826171875  | 39.990234375 |     1      |
  | 39.96826171875  | 39.979248046875  | 39.990234375 |     1      |
    | 39.979248046875 | 39.9847412109375 | 39.990234375 |     1      |

  - **经度解析 Lng：116.3063484933343**
  
    |     做区间      |       中值       |     右区间     | 二进制结果 |
    | :-------------: | :--------------: | :------------: | :--------: |
    |      -180       |        0         |      180       |     1      |
    |        0        |        90        |      180       |     1      |
    |       90        |       135        |      180       |     0      |
    |       90        |      112.5       |      135       |     1      |
    |      112.5      |      123.75      |      135       |     0      |
    |      112.5      |     118.125      |     123.75     |     0      |
    |      112.5      |     115.3125     |    118.125     |     1      |
    |    115.3125     |    116.71875     |    118.125     |     0      |
    |    115.3125     |    116.015625    |   116.71875    |     1      |
    |   116.015625    |   116.3671875    |   116.71875    |     0      |
    |   116.015625    |   116.19140625   |  116.3671875   |     1      |
    |  116.19140625   |  116.279296875   |  116.3671875   |     1      |
    |  116.279296875  |  116.3232421875  |  116.3671875   |     0      |
  |  116.279296875  | 116.30126953125  | 116.3232421875 |     1      |
    | 116.30126953125 | 116.312255859375 | 116.3232421875 |     0      |

  - **纬度 Lat** 二进制为：`1011 1000 1101 110`

  - **经度 Lng** 二进制为：`1101 0010 1011 010`
  
  - 按照 `经纬经纬（偶数经度，奇数纬度）...` 二进制重新组合为：`11100 11101 00100 01101 10110 11100`
  
    | 二进制 | 十进制 | Base 32 |
    | :----: | :----: | :-----: |
    | 11100  |   28   |    w    |
    | 11101  |   29   |    x    |
    | 00100  |   4    |    4    |
    | 01101  |   13   |    e    |
    | 10110  |   22   |    q    |
    | 11100  |   28   |    w    |
  
  -  **由此简单验证了 Geohash 基于 Z 阶曲线和 Base 32 算法的结论。**

### 8.3 Google S2 算法

- [Google’s S2 library](https://code.google.com/p/s2-geometry-library/) is a real treasure, not only due to its capabilities for spatial indexing but also because it is a library that was released more than 4 years ago and it didn’t get the attention it deserved.

- 上边介绍了如何使用  `Z 阶曲线` 来解决多维空间点索引问题。接下来介绍另外一种算法 **S2** 。

- **多维空间点索引需要解决 2 个问题**

  -   **如何把多维降维低维或者一维？**
  -   **一维曲线如何分形（采用哪一种图像作为基本的 Cell）？**

-   **空间填充曲线，将多维降为一维**

    -   **在数学分析中，有这样一个难题：能否用一条无限长的线，穿过任意维度空间里面的所有点？**

        -   Peano Curve（1890 年）
            -   取一个正方形并且把它分出九个相等的小正方形，然后从左下角的正方形开始至右上角的正方形结束，依次把小正方形的中心用线段连接起来；下一步把每个小正方形分成九个相等的正方形，然后上述方式把其中中心连接起来……将这种操作手续无限进行下去，最终得到的极限情况的曲线就被称作皮亚诺曲线。

        ![PeanoCurve](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChat1bf2f8b89674332c99f07727e0f33b2f.png)

    -   Hilbert Curve（一年后，希尔伯特做出了这条曲线）

        ![HilbertCurve](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/20200628224218.png)

-   **分形**

    -   多维空间降维以后，如何分形，也是一个问题。分形的方式有很多种，这里有一个[WIKI 分形列表](https://en.wikipedia.org/wiki/List_of_fractals_by_Hausdorff_dimension)，可以查看如何分形，以及每个分形的分形维数，即豪斯多夫分形维(Hausdorff fractals dimension)和拓扑维数。

    -   **Google S2 采用了正方形分形，如下图**

        ![Square-S2](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChatdb1c80b9b20990f096d7e86490b0b646.png)

-   **Google S2 Cell 精度**

    | Level |  Min Area   |  Max Area   | Average Area |     Units      | Random cell 1 (UK) min edge length | Random cell 1 (UK) max edge length | Random cell 2 (US) min edge length | Random cell 2 (US) max edge length | Number of cells |
    | :---: | :---------: | :---------: | :----------: | :------------: | :--------------------------------: | :--------------------------------: | :--------------------------------: | :--------------------------------: | :-------------: |
    |  00   | 85011012.19 | 85011012.19 | 85011012.19  | km<sup>2</sup> |              7842 km               |              7842 km               |              7842 km               |              7842 km               |        6        |
    |  01   | 21252753.05 | 21252753.05 | 21252753.05  | km<sup>2</sup> |              3921 km               |              5004 km               |              3921 km               |              5004 km               |       24        |
    |  02   | 4919708.23  | 6026521.16  |  5313188.26  | km<sup>2</sup> |              1825 km               |              2489 km               |              1825 km               |              2489 km               |       96        |
    |  03   | 1055377.48  | 1646455.50  |  1328297.07  | km<sup>2</sup> |               840 km               |              1167 km               |              1130 km               |              1310 km               |       384       |
    |  04   |  231564.06  |  413918.15  |  332074.27   | km<sup>2</sup> |               432 km               |               609 km               |               579 km               |               636 km               |      1536       |
    |  05   |  53798.67   |  104297.91  |   83018.57   | km<sup>2</sup> |               210 km               |               298 km               |               287 km               |               315 km               |       6k        |
    |  06   |  12948.81   |  26113.30   |   20754.64   | km<sup>2</sup> |               108 km               |               151 km               |               143 km               |               156 km               |       24k       |
    |  07   |   3175.44   |   6529.09   |   5188.66    | km<sup>2</sup> |               54 km                |               76 km                |               72 km                |               78 km                |       98k       |
    |  08   |   786.20    |   1632.45   |   1297.17    | km<sup>2</sup> |               27 km                |               38 km                |               36 km                |               39 km                |      393k       |
    |  09   |   195.59    |   408.12    |    324.29    | km<sup>2</sup> |               14 km                |               19 km                |               18 km                |               20 km                |      1573k      |
    |  10   |    48.78    |   102.03    |    81.07     | km<sup>2</sup> |                7 km                |                9 km                |                9 km                |               10 km                |       6M        |
    |  11   |    12.18    |    25.51    |    20.27     | km<sup>2</sup> |                3 km                |                5 km                |                4 km                |                5 km                |       25M       |
    |  12   |    3.04     |    6.38     |     5.07     | km<sup>2</sup> |               1699 m               |                2 km                |                2 km                |                2 km                |      100M       |
    |  13   |    0.76     |    1.59     |     1.27     | km<sup>2</sup> |               850 m                |               1185 m               |               1123 m               |               1225 m               |      402M       |
    |  14   |    0.19     |    0.40     |     0.32     | km<sup>2</sup> |               425 m                |               593 m                |               562 m                |               613 m                |      1610M      |
    |  15   |  47520.30   |  99638.93   |   79172.67   | m<sup>2</sup>  |               212 m                |               296 m                |               281 m                |               306 m                |       6B        |
    |  16   |  11880.08   |  24909.73   |   19793.17   | m<sup>2</sup>  |               106 m                |               148 m                |               140 m                |               153 m                |       25B       |
    |  17   |   2970.02   |   6227.43   |   4948.29    | m<sup>2</sup>  |                53 m                |                74 m                |                70 m                |                77 m                |      103B       |
    |  18   |   742.50    |   1556.86   |   1237.07    | m<sup>2</sup>  |                27 m                |                37 m                |                35 m                |                38 m                |      412B       |
    |  19   |   185.63    |   389.21    |    309.27    | m<sup>2</sup>  |                13 m                |                19 m                |                18 m                |                19 m                |      1649B      |
    |  20   |    46.41    |    97.30    |    77.32     | m<sup>2</sup>  |                7 m                 |                9 m                 |                9 m                 |                10 m                |       7T        |
    |  21   |    11.60    |    24.33    |    19.33     | m<sup>2</sup>  |                3 m                 |                5 m                 |                4 m                 |                5 m                 |       26T       |
    |  22   |    2.90     |    6.08     |     4.83     | m<sup>2</sup>  |               166 cm               |                2 m                 |                2 m                 |                2 m                 |      105T       |
    |  23   |    0.73     |    1.52     |     1.21     | m<sup>2</sup>  |               83 cm                |               116 cm               |               110 cm               |               120 cm               |      422T       |
    |  24   |    0.18     |    0.38     |     0.30     | m<sup>2</sup>  |               41 cm                |               58 cm                |               55 cm                |               60 cm                |      1689T      |
    |  25   |   453.19    |   950.23    |    755.05    | cm<sup>2</sup> |               21 cm                |               29 cm                |               27 cm                |               30 cm                |      7e15       |
    |  26   |   113.30    |   237.56    |    188.76    | cm<sup>2</sup> |               10 cm                |               14 cm                |               14 cm                |               15 cm                |      27e15      |
    |  27   |    28.32    |    59.39    |    47.19     | cm<sup>2</sup> |                5 cm                |                7 cm                |                7 cm                |                7 cm                |     108e15      |
    |  28   |    7.08     |    14.85    |    11.80     |      cm2       |                2 cm                |                4 cm                |                3 cm                |                4 cm                |     432e15      |
    |  29   |    1.77     |    3.71     |     2.95     |      cm2       |               12 mm                |               18 mm                |               17 mm                |               18 mm                |     1729e15     |
    |  30   |    0.44     |    0.93     |     0.74     |      cm2       |                6 mm                |                9 mm                |                8 mm                |                9 mm                |      7e18       |

    -   **S2 采用了 31 级精度，从  Level 00 - Level 30 分级范围之间差距比较均匀，不会像  Geohash 一样，层级之间分布不匀**
    -   **Level 00 将地球分为了 6 个正方形，到 Level 30 将地球分为 7e18 个正方形，比 Geohash 精度更为精准。**

-    **[Google S2 Java Libary](https://github.com/google/s2-geometry-library-java)**

-   **[About S2 Cells](https://s2geometry.io/devguide/s2cell_hierarchy)**

### 8.4 Geohash 存储方式

-   `Redis` 采用了 `Sorted Set` 存储地理位置。每一个 `member` 的 `score` 的大小是一个 **52 为的 Geohash 值（Double 精度为 52），由 26 位经度二进制和 26 位维度二进制交叉组合，最后转为十进制作为 score。 **

```bash
redis> GEOADD k1 116.3063484933343 39.981498991524006 yinke
1
redis> ZRANGE k1 0 -1 withscores
yinke
4069880514976585
redis> GEOHASH k1 yinke
wx4eqw7tn70

#########################
4069880514976585
十进制转换为二进制 
1110011101011000100011100110010111111001101101001001(对应的 Hash 值为：wx4eqw7tn70)
手动解析的六字符二进制
111001110100100011011011011100(对应的 Hash 值为：wx4eqw)
仔细对比发现前17位是一样的，这是因为 Geohash 精度不一样，Redis 自身采用 11 位，手动验证的时候为了简便取的是 6 位
```

### 8.5 Geohash 优缺点

-   **优点：**

    -   采用 `Z 阶曲线` 进行编码，将二维或者多维空间的所有点转换为一堆曲线，进而可以转换为一维数组结构，并具有局部保序性。在一维数组中可以采用二叉搜索树、B树、跳跃表或哈希表等处理数据。

-   **缺点：**

    -   虽然有局部保序性，但也有突变性。**相邻的 Z 首尾虽然两个点事相邻的，但是距离相隔很远。**

        ![Z-突变](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/20200628185553.png)

### 8.6 Geohash 常用命令

```bash
# 将指定的地理空间位置（纬度、经度、名称）添加到指定的key中。这些数据将会存储到 sorted set
GEOADD key longitude latitude member [longitude latitude member ...]

# 返回两个给定位置之间的距离。如果两个位置之间的其中一个不存在， 那么命令返回空值。
# unit 参数：不显示指定 unit 默认参数为 m
# m 单位 米
# km 单位 千米
# mi 单位 英里
# ft 单位 英尺
GEODIST key member1 member2 [unit]

# 返回一个或多个位置元素的 Geohash 表示。
GEOHASH key member [member ...]

# 从key里返回所有给定位置元素的位置（第一个元素是经度和第二个元素是纬度）
GEOPOS key member [member ...]

# 以给定的经纬度为中心，返回键包含的位置元素当中，与中心的距离不超过给定最大距离的所有位置元素。

# 额外参数：
# WITHDIST：在返回位置元素的同时，将位置元素与中心之间的距离也一并返回。距离的单位和用户给定的范围单位保持一致。
# WITHCOORD：将位置元素的经度和维度也一并返回。
# WITHHASH：以 52 位有符号整数的形式， 返回位置元素经过原始 geohash 编码的有序集合分值。这个选项主要用于底层应用或者调试，实际中的作用并不大。

# ASC: 根据中心的位置，按照从近到远的方式返回位置元素。
# DESC: 根据中心的位置，按照从远到近的方式返回位置元素。

# COUNT <count> 选项去获取前 N 个匹配元素，但是因为命令在内部可能会需要对所有被匹配的元素进行处理， 所以在对一个非常大的区域进行搜索时，即使只使用 COUNT 选项去获取少量元素，命令的执行速度也可能会非常慢。 但是从另一方面来说，使用 COUNT 选项去减少需要返回的元素数量，对于减少带宽来说仍然是非常有用的。

GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]

# 和 GEORADIUS 命令一样，都可以找出位于指定范围内的元素，但是 GEORADIUSBYMEMBER 的中心点是由给定的位置元素决定的，而不是像 GEORADIUS 那样，使用输入的经度和纬度来决定中心点。
# 指定成员的位置被用作查询的中心。
GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]
```

### 8.7 应用场景

-   地理相关应用
-   附近地理相关的所有应用

## 9. Stream

> 进入Redis 客户端使用     help @stream
>
> 示例：https://redis.io/commands#stream

**Stream 是 Redis 5.0 新增特性，和上述 8 中数据类型相比，也是数据结构最为复杂的一个。**

**每个消息流都包含一个 Rax 结构。以消息ID为 key，listpack 结构为 value 存储在 Rax 结构中。每个消息的具体信息存储在这个 listpack 中。**

![StreamStructure](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/StreamStructure.svg)

### 9.1 Stream 数据结构

-   **了解 Stream 首先需要了解其内部数据结构**

-   **Rax Tree（前缀树也成基数树）**

    -    [Wiki 关于 Radix Tree的图示](https://en.wikipedia.org/wiki/Radix_tree) 是字符串查找时，经常使用的一种数据结构，能够在一个字符串集合中快速查找到某个字符串。

        ![Radix-Tree](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/WeChat0e701d81d2e7013595cff8d2b959f319.png)

    -   **Redis Rax 结构由  3 部分组成**

        -   ```c
            typedef struct rax {
            	raxNode *head;
            	uint64_t numele;
            	uint64_t numnodes;
            }
            
            typedef struct raxNode {
            	uint32_t iskey:1;
            	uint32_t isnull:1;
            	uint32_t iscompr:1;
            	uint32_t size:29;    
            	unsigned char data[];    
            }
            
            raxNode 结构：
            +-------+--------+---------+------+------+
            | isKey | isNull | iscompr | size | data |
            +-------+--------+---------+------+------+
                
            raxNode 分为两类：
            1. 压缩节点：iscompr = 1 
               iskey = 1 和 isnull = 0 时，value-ptr 存在，否则 value-ptr 不存在
            
            以存储压缩节点 ABC 为例：
            
            +-------+--------+---------+------+---+---+---+-----+-------+----------+
            |isKey=1|isNull=0|iscompr=1|size=3| A | B | C | pad | C-ptr |value-ptr?|
            +-------+--------+---------+------+---+---+---+-----+-------+----------+
            
            2. 非压缩节点：iscompr = 0，Redis 中，只有不大于 2 个字符时才是非压缩节点
            
            +-------+--------+---------+------+---+---+-----+-------+-------+----------+
            |isKey=1|isNull=0|iscompr=0|size=2| X | Y | pad | X-ptr | Y-ptr |value-ptr?|
            +-------+--------+---------+------+---+---+-----+-------+-------+----------+
            
            ```

        -   **Header：** 指向头节点指针

            -   **iskey：** 表明当前节点是否包含一个 key，占用 **1** bit
            -   **isnull：** 表明当前 key 对应的 value 是否为空，占用 **1** bit
            -   **iscompr：**表明当前节点是否为压缩节点，占用 **1** bit
            -   **size：**为压缩节点压缩的字符串长度或者非压缩节点的子节点个数，占用 **29** bit
            -   **data：**包含填充字段，同时**存储了当前节点包含的字符串以及子节点的指针、key 对应的 value指针（Stream 具体消息内容）**

        -   **Numele：** 元素个数

        -   **Numnodes：** 节点个数

-   **Listpack（紧凑列表）由 4 部分组成**

    -   ```c
        +-----------+--------+----------+----------+------+----------+-----+
        |Total Bytes|Num Elem|  Entry1  |  Entry2  |  ... |  EntryN  | End |
        +-----------+--------+----------+----------+------+----------+-----+
                                /    \
                               /      \
                              /        \
                    +----------+---------+---------+
                    |  Encode  | Content | Backlen |
                    +----------+---------+---------+
        ```

    -   **Total Bytes：** 整个 `listpack` 空间大小，占用 **4** 个字节，每个 `listpack` 最多占用 **4294967295 Bytes**

    -   **Num Elem：** `listpack` 中元素个数，即 `Entry` 个数。占用  **2** 个字节（1个字符），即最大表示 **65535**，当超出时，仍然可以继续存储。

    -   **Entry：**每个具体元素，由 **3** 部分组成

        -   **Encode：** 元素的编码方式，占用 **1** 字节。
        -   **Content：**元素内容
        -   **Backlen：**记录 Entry 长度（Encode + Content 长度，不包含自身）

    -   **End：**`listpack` 结束标志，占用 **1** 字节，内容为 `0xFF`

### 9.2 Stream 组成部分（消息、生产者、消费者、消费组）

- **消息：** Stream 消息队列中的内容

  -   **消息 ID ：每个消息右唯一的一个消息 ID 并且严格递增**
      -   **Unix time（millionseconds）+ sequence**
          -   **格式：1593444536965-0** 
  -   **消息内容：（多个 k-v 结构）**
      -   `field string [field string ...]`

- **生产者：** 负责向 Stream 消息队列生产内容

  -   **命令：**`XADD key ID field string [field string ...]`

- **普通消费者：** 获取 Stream 消息队列中的内容，当消费者不归属消费组时，可以消费消息队列中任何消息。

  -   **命令：**`XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]`
      -   **非阻塞模式：（默认获取方式）**
          -   指定 ID 为 **0** 读取，获取消息队列中所有消息
          -   指定 ID 为**某一个消息ID**，获取该 ID 之后的所有消息
      -   **阻塞模式：（BLOCK 参数默认 millionseconds）**
          -   指定 ID 为 **0** ，并设置 `BLOCK` 时间，阻塞模式会立即获取消息队列中全部消息**（不推荐使用）**
          -   指定 ID 为 **$** ，并设置 `BLOCK` 时间，阻塞模式会在指定的 `阻塞时间` 内等待消息队列中的**新消息或阻塞时间内没有新消息结束阻塞**。当阻塞时间结束，该次阻塞模式结束，并不会等待第二次或更多次获取消息。

- **消费组：** `Stream` 内一个重要概念。允许消费者将一个 `Stream` 从逻辑上划分为不同的流，并让消费组下的消费者取处理组中`待处理状态（未消费）`消息。

  -   **消费组特点：**
      -   每个消费组通过组名称唯一标识，**每个消费组都可以消费该消息队列中的全部消息**，多个消费组之间**相互独立**。
      -   每个消费组可以有**多个消费者**，消费者通过名称唯一标识，消费者之间的关系是**竞争关系**，也就是说一个消息只能由该组的一个成员消费。
      -   组内成员消费消息后**需要确认**，每个消息组都有一个**待确认消息队列**（pending entry list, pel），用以维护该消费组**已经消费但没有确认**的消息。
      -   消费组中的每个成员也有一个**待确认消息队列**，维护着该消费者**已经消费尚未确认**的消息。
  -   **命令：**`XGROUP [CREATE key groupname id-or-$] [SETID key id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]`
      -   指定 ID 为 **0**，则获取消费组内**所有待处理消息**。
      -   指定 ID 为**某一个消息ID**，则获取该消息 ID 之后的**所有待处理消息**。
      -   指定 ID 为 **$**，则获取该组建立之后的**新消息**。

-   **组内消费者：**

    -   **命令：**`XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]`
        -   指定 ID 为 **0**，则获取消费组建立后的**所有待处理消息**。
        -   指定 ID 为 **>**，则获取消费组内**未被消费起始的消息**。**注意 ID 特殊符号是 >，不同于普通消费者的 $**
        -   **不支持指定消息 ID**
    
    ![Stream-msq](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Stream-msg.svg)

### 9.3 Stream 命令

-   `Stream 读写命令`
    -   **追加新元素到流的末尾**
        -   `XADD key ID field string [field string ...]`
    -   **获取流内的元素数量**
        -   `XLEN key`
    -   **访问流中元素，双向迭代**
        -   `start` 和 `end` 可以是 `特殊值减号 -(最小ID) 和加号 +(最大ID)` 也可以使指定 ID，获取范围是`左右开区间`。
        -   `COUNT count` 通过对流的迭代获取 `前N 个元素`。Stream 内部实现的是双向迭代器。
            -   正序访问（从小到大）`XRANGE key start end [COUNT count]`
            -   倒序访问（从大到小）`XREVRANGE key end start [COUNT count]`
    -   **以阻塞或非阻塞方式获取流元素，只能按照流开始的方向迭代，但是可以同时从多个流中迭代，每次迭代都需要获取上一次迭代返回的最后元素的 ID**
        -   `XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]`
            -   `COUNT count` 选项在 `XREAD` 后边是因为 `STREAMS` 选项是可变参数选项
            -   `key [key ...]` 和 `ID [ID ...]` 流和流内的ID必须按照顺序对应
    -   **对流进行修剪**
        -   `XTRIM key MAXLEN [~] count`
    -   **移除指定元素**
        -   `XDEL key ID [ID ...]`
-   `Stream 组概念相关命令`
    -   **创建 / 销毁 消费者组（组内消费者）**
        -   `XGROUP [CREATE key groupname id-or-$] [SETID key id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]`
    -   **读取消费者组中的消息**
        -   `XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]`
    -   **查看组内（组内消费者）待处理（已读取未确认）状态消息的相关信息**
        -   `XPENDING key group [start end count] [consumer]`
    -   **转移待处理（已读取未确认）状态消息的归属权**
        -   `XCLAIM key group consumer min-idle-time ID [ID ...] [IDLE ms] [TIME ms-unix-time] [RETRYCOUNT count] [force] [justid]`
    -   **将消息标记为已处理状态（确认消息）**
        -   ` XACK key group ID [ID ...]`
-   `Stream 查看消费组、消费者、流 命令`
    -   `XINFO [CONSUMERS key groupname] [GROUPS key] [STREAM key] [HELP]`

### 9.4 Stream 命令详解

#### 9.4.1 生产者与消费者模式

-   执行 `XADD` 命令

    -   `XADD mq * name boni` 

    -   `XADD mq * name lily age 18`

    -   `XADD mq * gender female married false address China`

    -   **原理**

        -   `Stream` 通过执行 `XADD` 命令，将一个带有 **指定ID（通常使用 * 采用 Unix time + Seq 方式）** 以及包含指定键值对的元素追加到流的末尾。如果**流不存在**，`Redis` 会先创建一个空白的流，然后将给定元素追加到流中。
        -   同一个流中的每个元素都可以包含一个或任意多个键值对。**元素内的键值对存储顺序是以用户创建顺序方式进行存储的。**
        -   流元素的 ID，从小到大依次存储在 `Rax Tree` 上。通常使用 `Redis` 自动生成的方式**（*）**,如果手动指定 ID，则该 ID 必须大于流内末尾的 ID。
            -   当指定 ID 为 **1000**，自动补全 `sequence` 并返回 **1000-1**。
            -   只能将新元素添加到末尾而不允许在数据结构的 `中间` 添加新元素，这也是**流与列表以及有序集合之间的一个显著区别**。

    -   **图示**

        ![Stream-ADD](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Stream-ADD命令.svg)



-   执行 `XADD MAXLEN [~] count` 命令，该命令效果同 `XTRIM`

    -   `XADD mq MAXLEN 3 * name lucy age 19`

    -   `XADD mq MAXLEN 3 * name Tom age 21`

    -   **原理**

        -   `MAXLEN` 限制流的长度，当添加的新元素超过指定的 `count` 时，会按照`先进先出` 的规则移除超过长度限制的元素。
        -   `[~]`  中括号是 `Redis` 命令的可选参数，如果使用  `~` 则会按照 `count` 的概数限制元素个数。比如 `MAXLEN ~ 100000`，则最后流内元素个数可能比 100000 多一些，也或许会少一些。**该参数不建议使用，实验添加了 20 个元素，使用 MAXLEN ~ 10 并未起到效果，可能跟数据量太小有关。**
        -   `count` 指定限制流的长度。

    -   **图示，最后保留了 3、4、5 三个元素，1 和 2 被移除**

        ![Stream-ADD-MAXLEN](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Stream-ADD-MAXLEN命令.svg)

-   执行 `XREAD` 命令

    -   `XADD mq * name boni` 和 `XADD mq * name lily` 并返回消息ID `1593515337049-0` 和 `1593515499000-0`
    -   `XREAD STREAMS mq 1593515337049-0`  可以获取该 ID 后的消息
    -   `XREAD STREAMS mq 0`  可以获取 `0-0` 起始以后的所有消息，`0 是 0-0 的简写`
    -   `XREAD STREAMS mq $` 不会获取任何消息，因为 `$` 配合的是 `BLOCK` 阻塞模式，非阻塞模式无意义。

-   执行 `XREAD BLOCK milliseconds` 命令

    -   **Client 1**

        -   `XREAD BLOCK 0 STREAMS mq $`

    -   **Client 2**

        -   `XREAD BLOCK 0 STREAMS mq $`

    -   **Client 3**

        -   `XADD mq * name boni`

    -   **结果：**

        -   `Client 1 和 2` 将进入阻塞状态。在此之后，如果在给定的时限内，有 `Client 3` 向 `流 mq` 推入新元素，那么 `Client 1 和 2` 的阻塞状态就会被解除，并返回被推入的元素。
        -   `$` 代表最新消息 ID，配合阻塞模式使用。

    -   **图示**

        ![Stream-READ-BLOCK](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Stream-READ-BLOCK.svg)

#### 9.4.2 生产者与消费者组模式

##### 9.4.2.1 管理消费组

- 执行 `XGROUP CREATE` 命令，三种 ID 方式读取流消息**（Client 2、3、4互相之间是独立的，只和  Clinet 1 交互）**

  - **Clinet 1**

    - `XADD mq * name lily `
    - `XADD mq * name boni`

  - **Client 2，指定 ID 为 0 从流的开始读取消息**

    - `XCREATE GROUP mq mqGroup1 0`

  - **Client 3，指定 ID 为固定流 ID 读取该 ID 以后的消息**

    - `XCREATE GROUP mq mqGroup2 20000-0`

  - **Clinet 4，指定 ID 为 $ 以组创建时流内的最后 ID 为起始读取消息**

    - `XCREATE GROUP mq mqGroup3 $`

  - **图示**

    ![Stream-Create-Group](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Stream-GROUP-CREATE命令.svg)

  - **命令演示**

    - **Client 1 和 Clinet 2 交互**

      - ```bash
        # 1. Clinet 1 生产消息
        Redis-1> XADD mq * name lily
        "1593533700757-0"
        Redis-1> XADD mq * name boni
        "1593533709161-0"
        
        # 2. Client 2 在流 mq 中建立消费组 mqGroup1 读取以流起始 ID 的所有消息
        Redis-2> XGROUP CREATE mq mqGroup1 0
        OK
        
        # 3. Client 2 在流 mq 中的消费组 mqGroup1 的消费者 consumerA 读取消息，读取了流起始到目前的所有消息
        Redis-2> XREADGROUP GROUP mqGroup1 consumerA STREAMS mq >
        1) 1) "mq" 
           2) 1) 1) "1593534016678-0"
                 2) 1) "name"
                    2) "lily"
              2) 1) "1593534017725-0"
                 2) 1) "name"
                    2) "boni"
                    
        # 此时如果 Clinet 1 再生产 3 条信息，Client 2 执行命令2，则会再次读取 3 条消息   
        ```

    - **Client 1 和 Client 3 交互**

      - ```bash
        # 1. Clinet 1 生产消息
        Redis-1> XADD mq * name lily
        "1593533700757-0"
        Redis-1> XADD mq * name boni
        "1593533709161-0"
        
        # 2. Client 3 在流 mq 中建立消费组 mqGroup1 以指定的 ID 为起始读取指定 ID 以后的新消息
        Redis-3> XGROUP CREATE mq mqGroup1 1593533709161-0
        OK
        
        # 3. Client 3 在流 mq 中的消费组 mqGroup2 的消费者 consumerA 读取消息，读取指定流ID 1593533709161-0 起始到目前的所有消息
        Redis-3> XREADGROUP GROUP mqGroup2 consumerA STREAMS mq >
        (nil) # 返回 nil，此时流 mq 内没有比 ID 1593533709161-0 大的消息，即没有新消息
        
        # 4. Clinet 1 生产 3 条新的消息
        Redis-1> XADD mq * name lucy
        "1593535028464-0"
        Redis-1> XADD mq * name Tom
        "1593535041603-0"
        Redis-1> XADD mq * name Jerry
        "1593535047183-0"
        
        # 5. Client 3 在流 mq 中的消费组 mqGroup2 的消费者 consumerA 读取消息，读取指定流ID 1593534017725-0 起始到目前的所有消息
        Redis-3> XREADGROUP GROUP mqGroup2 consumerA STREAMS mq >
        1) 1) "mq"
           2) 1) 1) "1593535028464-0"
                 2) 1) "name"
                    2) "lucy"
              2) 1) "1593535041603-0"
                 2) 1) "name"
                    2) "Tom"
              3) 1) "1593535047183-0"
                 2) 1) "name"
                    2) "Jerry"
        ```

    - **Client 1 和 Client 4 交互**

      - ```bash
        # 1. Clinet 1 生产消息
        Redis-1> XADD mq * name lily
        "1593533700757-0"
        Redis-1> XADD mq * name boni
        "1593533709161-0"
        
        # 2. Client 4 在流 mq 中建立消费组 mqGroup3 以组创建时流内最后 ID 为起始读取以后的新消息
        Redis-4> XGROUP CREATE mq mqGroup3 $
        OK
        
        # 3. Client 4 在流 mq 中的消费组 mqGroup3 的消费者 consumerA 读取消息，读取组创建时流内最后 ID 为起始的新消息
        Redis-3> XREADGROUP GROUP mqGroup2 consumerA STREAMS mq >
        (nil) # 返回 nil，此时流 mq 内没有新消息
        
        # 4. Clinet 1 生产 3 条新的消息
        Redis-1> XADD mq * name lucy
        "1593535028464-0"
        Redis-1> XADD mq * name Tom
        "1593535041603-0"
        Redis-1> XADD mq * name Jerry
        "1593535047183-0"
        
        # 5. Client 4 在流 mq 中的消费组 mqGroup3 的消费者 consumerA 读取消息，读取组创建时流内最后 ID 为起始的新消息，并获取 3 条信息
        Redis-3> XREADGROUP GROUP mqGroup2 consumerA STREAMS mq >
        1) 1) "mq"
           2) 1) 1) "1593535028464-0"
                 2) 1) "name"
                    2) "lucy"
              2) 1) "1593535041603-0"
                 2) 1) "name"
                    2) "Tom"
              3) 1) "1593535047183-0"
                 2) 1) "name"
                    2) "Jerry"
        ```

- 执行 `XGROUP DESTROY` 命令**（删除消费组，该命令慎用）**

  - `XGROUP DESTROY mq mqGroup`
    
    - 删除消费组，级联删除组内消费者及消费者的待处理消息（**即 XPENDING 命令内涉及到该消费组内的消费者待处理的消息会被删除）**。
    
  - **命令演示**

      - ```bash
          # 1. Clinet 2 创建消费组 mq
          Redis-2> XGROUP CREATE mq mqGroup $
          OK
          
          # 2. Client 1 向流 mq 生产消息
          Redis-1> XADD mq * name boni
          "1593581940746-0"
          
          # 2. Client 2 创建组后，执行该命令进入阻塞状态，当 Client 1 生产消息后，Client 2 接触阻塞
          Redis-2> XREADGROUP GROUP mqGroup consumerA BLOCK 0 STREAMS mq >
          1) 1) "mq"
             2) 1) 1) "1593581940746-0"
                   2) 1) "name"
                      2) "boni"
          (4.35s)
          
          # 3. 使用 PENDING 命令，查看 流 mq 的消费组 mqGroup 情况
          Redis-2> XPENDING mq mqGroup
          1) (integer) 1				# 1 个已读取但未处理的消息
          2) "1593581940746-0"		# 起始 ID
          3) "1593581940746-0"		# 结束 ID
          4) 1) 1) "consumerA"		# 消费者 A 有 1 个待处理消息
                2) "1"
          
          # 3. 使用 XINFO 命令，查看 流 mq 的消费组情况
          Redis-2> XINFO GROUPS mq
          1) 1) "name"			
             2) "mqGroup"
             3) "consumers"
             4) (integer) 1			# 1 个消费者
             5) "pending"				
             6) (integer) 1			# 1 个待处理消息
             7) "last-delivered-id"  
             8) "1593581940746-0"     
          
          # 4. 使用 DESTROY 删除消费组 mqGroup，该命令会级联删除消费组内 consumerA 的待处理消息
          Redis-2> XGROUP DESTROY mq mqGroup
          (integer) 1
          
          # 5. 再次使用 XINFO 命令，查看 流 mq 的消费组情况
          Redis-2> XINFO GROUPS mq
          (empty list or set)         # 没有数据
          
          # 5. 再次使用 PENDING 命令，查看 流 mq 的消费组 mqGroup 情况
          Redis-2> XPENDING mq mqGroup
          (error) NOGROUP No such key 'mq' or consumer group 'mqGroup'    # 消费组已经不存在
          ```

- 执行 `XGROUP DELCONSUMER` 命令**（删除组内消费者，该命令慎用）**

  - `XGROUP DELCONSUMER mq mqGroup1 consumerA`
    - 删除消费者，会同时删除消费者的待处理消息（**即 XPENDING 命令会删除该消费者待处理的消息）**。

- 执行 `XGROUP SETID` 命令**（修改消费组的最后传递消息 ID）**

  - `XGROUP SETID mq mqGroup3 30000-0`
    
    - 如果消费组 mqGroup3 已经消费到 50000-0 使用该命令，则当再次使用 `XREADGROUP` 命令时，会读取 30000-0 以后的消息，并且使一次性获取。
    
  - **命令演示**
  
      - ```bash
          # 1. Clinet 1 向流 mq 生产消息
          Redis-1> XADD mq * name lily
          "1593585640284-0"
          Redis-1> XADD mq * name boni
          "1593585643751-0"
          Redis-1> XADD mq * name lucy
          "1593585702814-0"
          Redis-1> XADD mq * name Tom
          "1593585720628-0"
          
          # 2. Clinet 2 创建消费组 mqGroup
          Redis-2> XGROUP CREATE mq mqGroup $
          OK
          
          # 3. Clinet 1 在 Client 2 阻塞时，生产消息
          Redis-1> XADD mq * name Jerrry
          "1593585727079-0"
          
          # 3. Client 2 在流 mq 中的消费组 mqGroup 的消费者 consumerA 使用阻塞状态读取消息
          Redis-2> XREADGROUP GROUP mqGroup consumerA BLOCK 0 STREAMS mq >
          1) 1) "mq"
             2) 1) 1) "1593585727079-0"
                   2) 1) "name"
                      2) "Jerry"
          
          # 4. Clinet 2 执行 SETID 命令，是消费组 mqGroup 下次读取消息从指定 ID 读取消息
          Redis-2> XGROUP SETID mq mqGroup 1593585702814-0
          OK
          
          # 5. Clinet 2 再次执行第三个步骤，获取 ID 为 1593585702814-0 后边的 2 条消息
          Redis-2> XREADGROUP GROUP mqGroup consumerA BLOCK 0 STREAMS mq >
          1) 1) "mq"
             2) 1) 1) "1593585720628-0"
                   2) 1) "name"
                      2) "Tom"
                2) 1) "1593585727079-0"
                   2) 1) "name"
                      2) "Jerrry"
          ```
  
  - **图示**
  
      ![Stream-Group-SETID](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Stream-命令SETID.svg)

##### 9.4.2.2 管理消费者

-   执行 `XREADGROUP GROUP`
    -   `Redis` 没有创建组内消费者的命令，执行该命令时，组内消费者会自动创建。
    -   `XREADGROUP GROUP` 指定流最后传递 ID 不同于 `XREAD` 的 `$`，此处使用  `>` 符号。

##### 9.4.2.3 显示待处理消息的相关信息（Stream 使用 Pending 列表记录读取但未处理完毕的消息）

-   执行 `XPENDING` 命令

    -   `XPENDING key group` 查看消费组内的消息 
    -   `XPENDING key group [start end count]` 查看消费组内消息，并指定起始、结束、数量
    -   `XPENDING key group [start end count] [consumer]` 查看消费组内消费者消息，并指定起始、结束、数量**（当使用  consumer 参数时， start end count 参数必须有）**

-   **命令演示**

    -   ```bash
        # 1. Client 1 生产消息
        Redis-1> XADD mq * name lily
        "1593588988257-0"
        Redis-1> XADD mq * name boni
        "1593588996656-0"
        
        # 2. Clinet 2 创建消费组，以流起始 ID 读取消息
        Redis-2> XGROUP CREATE mq mqGroup 0
        OK
        
        # 3. Clinet 2 消费组 mqGroup 内消费者 consumerA 读取流 mq 消息
        127.0.0.1:6379> XREADGROUP GROUP mqGroup consumerA STREAMS mq >
        1) 1) "mq"
           2) 1) 1) "1593588988257-0"
                 2) 1) "name"
                    2) "lily"
              2) 1) "1593588996656-0"
                 2) 1) "name"
                    2) "boni
                    
        # 4. Client 2 查询消费组 mqGroup 的 PENDING 情况
        Redis-2> XPENDING mq mqGroup
        1) (integer) 2				# 2条已读取但未处理的消息
        2) "1593588988257-0"		# 起始 ID
        3) "1593588996656-0"		# 结束 ID
        4) 1) 1) "consumerA"		# 消费者 consumerA 有 2 条未处理消息
              2) "2"
              
        # 5. Clinet 2 查询一定数量的消费组 mqGroup 的 PENDING 情况      
        Redis-2> XPENDING mq mqGroup - + 10
        1) 1) "1593588988257-0"		# 消息 ID
           2) "consumerA"			# 消费者 consumerA
           3) (integer) 2166070		# 从读取到现在经理了 2166070 ms
           4) (integer) 1			# 该消息被消费了 1 次		
        2) 1) "1593588996656-0"
           2) "consumerA"
           3) (integer) 2166070
           4) (integer) 1
           
        # 6. Client 2 查询一定数量的消费组 mqGroup 消费者 consumer A的 PENDING 情况      
        Redis-2> XPENDING mq mqGroup - + 10 consumerA
        1) 1) "1593588988257-0"		# 消息 ID
           2) "consumerA"			# 消费者 consumerA
           3) (integer) 2495859		# 从读取到现在经理了 2166070 ms	
           4) (integer) 1			# 该消息被消费了 1 次		
        2) 1) "1593588996656-0"
           2) "consumerA"
           3) (integer) 2495859
           4) (integer) 1   
        ```

##### 9.4.2.4 确认消息（并未在 Pending 列表删除，只是做了标记处理）

-   执行 `XACK` 命令，该命令会从消费者的 `PENDING` 队列标记该消息为`已处理`。之后执行 `XREADGROUP` 也不会在读取该消息。

-   **命令演示**

    -   ```bash
        # 使用 9.4.2.3 已有数据，直接执行 XACK 命令
        # 7. Client 2 确认 ID 1593588988257-0 消息
        Redis-2> XACK mq mqGroup 1593588988257-0
        (integer) 1   				# 处理的数量
        
        # 8. Client 2 再次查询消费者 consumerA 的待处理消息已经只有一条信息了。
        Redis-2> XPENDING mq mqGroup - + 10 consumerA
        1) 1) "1593588996656-0"
           2) "consumerA"
           3) (integer) 3086006
           4) (integer) 1
        ```

##### 9.4.2.5 转移消息

-   执行 `XCLAIM` 命令

-   **命令演示** 

    -   ```bash
        # 9. Clinet 2 查询消费者 consumerA 的 Pending 列表
        Redis-2> XPENDING mq mqGroup - + 10 consumerA
        1) 1) "1593588996656-0"
           2) "consumerA"
           3) (integer) 5341384
           4) (integer) 1
           
        # 10. Client 2 将消息 ID 为 1593588996656-0 并且读取时长超过 3000 ms 的消息转移给消费者 consumerB
        Redis-2> XCLAIM mq mqGroup consumerB 300000 1593588996656-0
        1) 1) "1593588996656-0"
           2) 1) "name"
              2) "boni"
        
        # 11. Clinet 2 再次查询消费者 consumerA 的待处理消息为空
        Redis-2> XPENDING mq mqGroup - + 10 consumerA
        (empty list or set)
        
        # 12. Clinet 2 再次查询消费者 consumerB 的待处理消息，返回了转移过来的消息
        Redis-2> XPENDING mq mqGroup - + 10 consumerB
        1) 1) "1593588996656-0"
           2) "consumerB"
           3) (integer) 7283
           4) (integer) 1
        ```

##### 9.4.2.6 删除消息（只删除 Stream 的消息列表，不会级联删除 Pending 相关数据，但是可以执行 XACK 清除）

-   执行 `XDEL` 命令，删除的 Stream 队列内的消息。

-   **命令演示**

    -   ```bash
        # 13. Clinet 1 再次添加 2 条信息
        Redis-1> XADD mq * name lucy
        "1593597231088-0"
        Redis-1> XADD mq * name Tom
        "1593597353678-0"
        
        # 14. Clinet 2 消费组 mqGroup 的消费者 consumerA 消费 2 条新的消息
        Redis-2> XREADGROUP GROUP mqGroup consumerA STREAMS mq >
        1) 1) "mq"
           2) 1) 1) "1593597231088-0"
                 2) 1) "name"
                    2) "lucy"
        Redis-2> XREADGROUP GROUP mqGroup consumerA STREAMS mq >
        1) 1) "mq"
           2) 1) 1) "1593597353678-0"
                 2) 1) "name"
                    2) "Tom"
        
        # 15. Client 2 查询流 mq  的所有消息内容，共 4 条
        Redis-2> XRANGE mq - +
        1) 1) "1593588988257-0"
           2) 1) "name"
              2) "lily"
        2) 1) "1593588996656-0"
           2) 1) "name"
              2) "boni"
        3) 1) "1593597231088-0"
           2) 1) "name"
              2) "lucy"
        4) 1) "1593597353678-0"
           2) 1) "name"
              2) "Tom"
        
        # 16. Clinet 2 查询 consumerA 的待处理消息（此时 conmserA 已经读取了 3、4 两条消息）
        Redis-2> XPENDING mq mqGroup - + 10 consumerA
        1) 1) "1593597231088-0"
           2) "consumerA"
           3) (integer) 787325
           4) (integer) 1
        2) 1) "1593597353678-0"
           2) "consumerA"
           3) (integer) 772945
           4) (integer) 1
        
        # 17. Clinet 2 删除 ID 为 1593597353678-0  的消息      
        Redis-2> XDEL mq 1593597353678-0
        (integer) 1
        
        # 18. Clinet 2 再次查询流 mq 的消息内容，变成了 3 条
        Redis-2> XRANGE mq - +
        1) 1) "1593588988257-0"
           2) 1) "name"
              2) "lily"
        2) 1) "1593588996656-0"
           2) 1) "name"
              2) "boni"
        3) 1) "1593597231088-0"
           2) 1) "name"
              2) "lucy"
              
        # 19. Clinet 2 当删除了流内的消息，再次查询 consuerA 的 Pending 列表，发现已删除的 1593597353678-0 还在，证明 XDEL 命令并不会级联删除 Pending 列表数据
        Redis-2> XPENDING mq mqGroup - + 10 consumerA
        1) 1) "1593597231088-0"
           2) "consumerA"
           3) (integer) 787325
           4) (integer) 1
        2) 1) "1593597353678-0"				# Pending 列表还存在已经删除的消息 ID
           2) "consumerA"
           3) (integer) 772945
           4) (integer) 1
        ```

##### 9.4.2.7 信息监控

-   执行 `XINFO` 命令

    -   `XINFO STREAM key`
    -   `XINFO GROUPS key`
    -   `XINFO CONSUMERS key groupName`

-   **命令演示**

    -   ```bash
        # Client 1 查询流队列消息
        Redis-1> XINFO STREAM mq
         1) "length"
         2) (integer) 2
         3) "radix-tree-keys"
         4) (integer) 1
         5) "radix-tree-nodes"
         6) (integer) 2
         7) "groups"
         8) (integer) 1
         9) "last-generated-id"
        10) "1593597353678-0"
        11) "first-entry"
        12) 1) "1593588988257-0"
            2) 1) "name"
               2) "lily"
        13) "last-entry"
        14) 1) "1593597231088-0"
            2) 1) "name"
               2) "lucy"
               
        # Clinet 1 查询消费组消息
        Redis-1> XINFO GROUPS mq
        1) 1) "name"
           2) "mqGroup"
           3) "consumers"
           4) (integer) 2
           5) "pending"
           6) (integer) 2
           7) "last-delivered-id"
           8) "1593597353678-0"
           
        # Client 1 查看消费组内的消费者消息
        Redis-1> XINFO CONSUMERS mq mqGroup
        1) 1) "name"
           2) "consumerA"
           3) "pending"
           4) (integer) 2
           5) "idle"
           6) (integer) 1119879
        2) 1) "name"
           2) "consumerB"
           3) "pending"
           4) (integer) 0
           5) "idle"
           6) (integer) 1578715
        ```

### 9.5 Stream ID 的 5 种场景

| ID                          | XREAD                                   | XRANGE / XREVRANGE | XGROUP                   | XREADGROUP                                                   | XPENDING       | XCLAIM                             | XDEL     |
| --------------------------- | --------------------------------------- | ------------------ | ------------------------ | ------------------------------------------------------------ | -------------- | ---------------------------------- | -------- |
| **0-0 简写 0**              | 从流开始 ID 读取                        | 有效               | 组创建时以流起始 ID 读取 | 以组创建时的最后消息 ID 为起始读取                           | 作用等同于 `-` | **无效**                           | **无效** |
| **指定 streamID**           | 从指定消息 ID 读取                      | 有效               | 组创建时以指定 ID 读取   | 以指定消息 ID 为起始读取                                     | 有效           | 有效，只针对指定 ID 转移消息归属权 | 有效     |
| **$**                       | 配合 `BLOCK` 阻塞命令，从流最后 ID 读取 | **无效**           | 组创建时以流最后 ID 读取 | **无效**                                                     | **无效**       | **无效**                           | **无效** |
| **>**                       | **无效**                                | **无效**           | **无效**                 | 以流内待处理的最后消息 ID 为起始读取`（如果流内有 3 条消息未被读取，该命令会读取 3 条）` | **无效**       | **无效**                           | **无效** |
| **-（最小ID） +（最大ID）** | **无效**                                | 有效               | **无效**                 | **无效**                                                     | 有效           | **无效**                           | **无效** |

### 9.6 Redis 内部实现的消息队区别

- **Redis 5.0 版本之前的实现方式**
    - **List** 
        - 列表实现的消息队列虽然可以快速地将新消息追加到列表的末尾，但因为列表为线性结构，所以程序如果想要查找包含指定数据的元素，或者进行范围查找，就需要遍历整个列表。
    - **Sorted Set**
        - 有序集合虽然可以有效地进行范围查找，但缺少列表和发布与订阅提供的阻塞弹出原语，这使得程序无法使用有序集合去实现可阻塞的消息弹出操作。
    - **Pub / Sub**
        -  发布与订阅虽然拥有将消息传递给多个客户端的能力，并且也拥有相应的阻塞弹出原语，但发布与订阅的“发送即忘（fire and forget）”策略会导致离线的客户端丢失消息，所以它是无法实现可靠的消息队列的。
- 无论是`列表`、`有序集合`还是`发布与订阅`，它们的元素都只能是`单个值`。换句话说，如果用户想要用这些数据结构实现的消息队列传递多项信息，那么必须使用 `JSON` 之类的序列化格式来将多项信息打包存储到单个元素中，然后再在取出元素之后进行相应的反序列化操作。
- **Stream**
    - `Redis Stream` 的出现解决了上述提到的所有问题，它是上述 **3** 种数据结构的综合体，具备它们各自的所有优点以及特点，是使用 `Redis` 实现消息队列应用的最佳选择。流是一个包含零个或任意多个流元素的`有序队列`，队列中的每个元素都包含一个 ID 和任意多个键值对，这些元素会根据 ID 的大小在流中有序地进行排列。
    - `Stream` 基于 `Pull` 模式，消费者需要主动拉取消息，根据消费能力进行消费。

### 9.6 Stream 应用场景

-   消息队列相关的应用场景
    -   即时通讯 - 聊天室等
    -   业务解耦