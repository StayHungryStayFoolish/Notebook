# Redis 的 9 种数据类型

## Redis 数据类型

## 1.String

>   进入Redis 客户端使用  help @string 查看命令
>
>   示例：https://redis.io/commands#string

**String 最基本的键值对数据类型，一种 key 和 value 可以是 String，也可以是普通的文字、图片、视频等二进制数据**

### 1.1 二进制安全

-   计算机存储的最小单位是 `Bit（比特，也成位）`，一个`Byte（字节）`。1 Byte = 8 bit。每个 Bit 用 `0` 或 `1` 来表示。`Character（字符）`通过不同的字符编码（ASCII、Unicode、UTF-8 、GBK等）由其指定固定的字节来表示。
-   `二进制安全` 是一个`输入`和`输出`以`字节`为单位的流，当数据`输入`时，不会对数据进行任何`限制`、`过滤`等（*例如：C 语言使用长度 N + 1 的方式表示字符串，N 为字符串，1 表示字符数组最后一个元素以'\0' 结尾，当读取到以 '\0' 结尾时结束*）。即无论`字符`以任何编码形式，最终存储的只是`字节`，保证了`输入`时的原始数据。存储和读取的双方只需约定好编码集就可以获取数据内容，具有跨平台、跨语言、防止数据类型溢出等优点。

### 1.2 Redis 的 SDS

-   C 语言的 `char`存储方式

    -   ![C char](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/c-char.png)
        -   以 `\0` 结尾，当获取`char`的`length`需要遍历数组直到空字符（**\0**）停止，复杂度为 **0(N)**
        -   `char`本身不记录长度，容易产生缓冲区溢出。

-   **SDS** 结构

    ```c
    typedef char *sds;
    
    struct sdshdr {
        // buf 已占用长度
        int len;
        // buf 剩余可用长度
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

-   通过 `len` 属性， `sdshdr` 可以实现复杂度为 θ(1)θ(1) 的长度计算操作。


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

## 6. BitMap    

**Bitmap(二进制位)属于 String 内的一种数据结构，因其应用场景较多，故单独拿出来。**

>   进入Redis 客户端使用  help @string 查看命令
>
>   示例：https://redis.io/commands#string

### 6.1 Bitmap 图解 (Bitmap 正规书写，图中 BitMap 书写错误)

-   **使用一个 `bit` 标记一个元素对应的`value`，`Key`即是该元素。**
  
  -   存储一个数组`[2,4,7]`
    
  	-   1.  计算机分配一个`byte`，初始化为8个为`0`的`bit`。
		-   2.  根据数组给定的值，在对应的`offset`将`bit`的值修改为`1`标记该元素。
        

  ![BitMap-Array](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap.jpg)
  
  -   增加一个元素`5`，使数组变为`[2,4,5,7]`
	

  ![BitMap-5](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-add.jpg)
      

  -   删除一个元素`4`，使数组变为`[2，7]`
	
  	-   **注意进行与运算时，只有 offset 为 4 的 bit 为 0，其余 bit 都是 1**


  ![BitMap-4](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-del.jpg)


  - 增加一个元素`20`,使数据变为`[2,4,7,20]`。
    
    
      	- 如果现有数组要增加元素，并且元素大于7，则再分配字节，并且`offset`仍然`从右向左`依次标记。例如增加20，则分配三个字节，分别是`buf[0]`、`buf[1]`、`buf[2]`，在`buf[2]`的`offset`对应元素标记为`1`。其中`buf[1]`的所有元素肯定为`0`。

  ![BitMap-add](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/bitmap-capacity.jpg)

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

        -   [二进制计算器](https://cn.calcuworld.com/二进制计算器) （该网站计算的值如果不够8位，需要在高位`左侧`补齐0）

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
        
        -   3.根据 `byte` 和 `bit` 值，在位数组 `bitarray` 中定位 `offset` 指定的二进制位，返回该位上的值。
        
        -   `GETBIT <bitarray> 3`
        
            ![GETBIT-3](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/GETBIT-3.jpg)
        
        -   `GETBIT <bitarray> 10`
        
    
    ![GETBIT-10](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/GETBIT-10.jpg)
    
    -   `SETBIT <bitarray> <offset> <value>` 命令实现，复杂度为`O(1)`
      -   1.  **len = ( offset ÷ 8 ) + 1**
        
            -   len 值记录了保存`offset`指定的二进制位需要多少字节（计算 bug[] ）
          
      -   2. 检查 `bitarray` 键保存的位数组**(sdshdr)**的长度是否小于 len。
        
            -   2.1 如果小于 len，则需要扩展字节。`sdshdr` 空间预分配策略会额外多分配 `len` 个字节的未使用空间，再加上为保存空字符而额外分配的1字节，及扩展后的字节为：**( len × 2 ) + 1**
          
      -   3.  **byte = offset ÷ 8 并向下取整**
        
            -   byte 值记录了`offset` 指定的二进制位保存在位数组的哪个字节。
          
      -   4. **bit = ( offset mod 8 ) + 1**
        
            -   bit 值记录了`offset`指定的二进制位中字节的第几个位置（**不是索引，是位置**）。
          
      -   5. 根据 **byte** 和 **bit** 值，在 `bitarray` 键保存的位数组中定位`offset`指定的二进制位。
        
            -   首先将指定的二进制位上当前值保存在 `oldvalue` 变量。
            -   然后将新值 `value` 设置为二进制位的值。
          
      -   6.返回 `oldvalue` 变量值。
        
            -   `SETBIT <bitarray> 1 1` 无需扩展字节
            
                ![SETBIT-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIT-1.jpg)
              
                ![SETBIE-1-1](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/SETBIT-1-1.jpg)
              
            -   `SETBIT <bitarray> 12 1`需扩展字节
            
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

`HyperLogLog` 是一个专门为了计算集合的基数而创建的概率算法，对于一个给定的集合，`HyperLogLog` 可以计算出这个集合的近似基数：近似基数并非集合的实际基数，它可能会比实际的基数小一点或者大一点，但是估算基数和实际基数之间的误差会处于一个合理的范围之内，因此那些不需要知道实际基数或者因为条件限制而无法计算出实际基数的程序就可以把这个近似基数当作集合的基数来使用。

`HyperLogLog` 的优点在于它计算近似基数所需的内存并不会因为集合的大小而改变，无论集合包含的元素有多少个，`HyperLogLog` 进行计算所需的内存总是固定的，并且是非常少的。
具体到实现上，`Redis` 的每个 `HyperLogLog` 只需要使用 `12KB` 内存空间，就可以对接近：`264` 个元素进行计数，而算法的标准误差仅为 `0.81%`，因此它计算出的近似基数是相当可信的。

### 7.1 HyperLogLog 算法

-   **基数计数基本概念**
  
    -   在数学上，**基数或势**，即集合中包含的元素的 `个数`。有限集合的基数，其意义与日常用语中的 `基数` 相同，例如 `{a,b,c}` 的基数是 `3`。
    -   **基数计数(Cardinality Counting)**通常用来统计一个集合中不重复的元素个数，例如统计某个网站的UV，或者用户搜索网站的关键词数量。数据分析、网络监控及数据库优化等领域都会涉及到基数计数的需求。 要实现基数计数，最简单的做法是记录集合中所有不重复的元素集合 **S<sub>u</sub>**，当新来一个元素 **x<sub>i</sub>**，若 **S<sub>u</sub>** 中不包含元素 **x<sub>i</sub>**，则将 **x<sub>i</sub>** 加入 **S<sub>u</sub>**，否则不加入，计数值就是 **S<sub>u</sub>** 的元素数量。这种做法存在两个问题：
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
  
-   **存储过程：**

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

### 7.4 HyperLogLog 常用命令

```bash
# 添加一个或多个元素（自动去重）
PFADD key element [element ...]

# 查看一个或多个 key 内包含的元素近似基数
PFCOUNT key [key ...]

# 合并多个 HyperLogLog 为一个，并保存到 destkey 中。（去重多个 HyperLogLog 内元素）
PFMERGE destkey sourcekey [sourcekey ...]
```

### 7.4 HyperLogLog 应用场景

-   大数据统计月活、日活
-   数据量不要求精确，允许少许误差的都可以，比 `Bitmap` 更节省内存

## 8. Geo

>   进入Redis 客户端使用     help @geo
>
>   示例：https://redis.io/commands#geo

## 9. Stream

```

```
