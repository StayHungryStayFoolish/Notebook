# Redis 

## Redis Single Install

-   Docker 

    ```bash
    # 拉取镜像
    docker pull redis:5.0.4
    # 启动镜像并指定密码
    docker run --name redis -d -p 6379:6379 redis --requirepass "password"
    # 登录 Redis 容器
    docker exec -it redis bash
    # 启动客户端
    redic-cli -a password
    ```

## Redis Cluster Install

-   Env

    -   Mac OS
    -   Docker 

-   Install

    -   docker 文件下执行 `docker-compose build`,然后执行 `docker-compose up`.

        `* Background AOF rewrite finished successfully` 表示集群启动成功.

-   Link

    [Mac上最简单明了的利用Docker搭建Redis集群](https://juejin.im/post/5cbd3c435188250a8b7cf55e)       

## Lettuce 

[Lettuce](https://lettuce.io/core/release/reference/#overview)

[Lettuce GitHub](https://github.com/lettuce-io/lettuce-core/wiki/About-Lettuce)

## Redis 数据结构

### 1.**String**

>   进入Redis 客户端使用  help @string 查看命令

#### 1.1二进制安全

-   计算机存储的最小单位是 `Bit（比特，也成位）`，一个`Byte（字节）`。1 Byte = 8 bit。每个 Bit 用 `0` 或 `1` 来表示。`Character（字符）`通过不同的字符编码（ASCII、Unicode、UTF-8 、GBK等）由其指定固定的字节来表示。
-   `二进制安全` 是一个`输入`和`输出`以`字节`为单位的流，当数据`输入`时，不会对数据进行任何`限制`、`过滤`等（*例如：C 语言使用长度 N + 1 的方式表示字符串，N 为字符串，1 表示字符数组最后一个元素以'\0' 结尾，当读取到以 '\0' 结尾时结束*）。即无论`字符`以任何编码形式，最终存储的只是`字节`，保证了`输入`时的原始数据。存储和读取的双方只需约定好编码集就可以获取数据内容，具有跨平台、跨语言、防止数据类型溢出等优点。

#### 1.2Redis 的 SDS

-   C 语言的 `char`存储方式

    -   ![C char](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/c-char.png?raw=true)
        -   以 `\0` 结尾，当获取`char`的`length`需要遍历数组直到空字符（**\0**）停止，复杂度为 **0(N)**
        -   `char`本身不记录长度，容易产生缓冲区溢出。

-   **SDS** 结构

    ```c#
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

       

#### 1.3字符串操作

-   常用命令

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

-   **使用场景**

    -   传统项目同步 session
    -   数据缓存（信息缓存、图片缓存、文件缓存）
    -   锁

#### 1.4数值操作

-   `Redis` 存储以`k-v`结构存储，`v`默认是字符串值，没有专用的`整数`和`浮点`类型。如果一个字符串值可以被`Redis内部`解释为`十进制64位有符号的整数、128位有符号的浮点数`时（Redis 内部有一个`RedisObject`的结构体会记录数据类型和编码格式等），则可以执行数值计算操作。

-   常用命令

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

-   **使用场景**

    -   计数器（统计访问次数）
    -   限速器（防止资源滥用）

#### 1.5二进制位（BitMap）

-   **BitMap**

    -   **使用一个 `bit` 标记一个元素对应的`value`，`Key`即是该元素。**
        -   存储一个数组`[2,4,7]`
            -   1.  计算机分配一个`byte`，初始化为8个为`0`的`bit`。
                2.  根据数组给定的值，在对应的`offset`将`bit`的值修改为`1`标记该元素。
            -   ![BitMap-Array](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap.jpg?raw=true)
        -   增加一个元素`5`，使数组变为`[2,4,5,7]`
            -   ![BitMap-5](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-add.jpg?raw=true)
        -   删除一个元素`4`，使数组变为`[2，7]`
            -   ![BitMap-4](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-del.jpg?raw=true)
                -   **注意进行与运算时，只有 offset 为 4 的 bit 为 0，其余 bit 都是 1**
        -   增加一个元素`20`,使数据变为`[2,4,7,20]`。
            -   如果现有数组要增加元素，并且元素大于7，则再分配字节，并且`offset`仍然`从右向左`依次标记。例如增加20，则分配三个字节，分别是`buf[0]`、`buf[1]`、`buf[2]`，在`buf[2]`的`offset`对应元素标记为`1`。其中`buf[1]`的所有元素肯定为`0`。
            -   ![BitMap-add](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-capacity.jpg?raw=true)

-   **常用命令**

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

            ![ascii](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/ascii.jpg?raw=true)

        -   [二进制计算器]([https://cn.calcuworld.com/%E4%BA%8C%E8%BF%9B%E5%88%B6%E8%AE%A1%E7%AE%97%E5%99%A8](https://cn.calcuworld.com/二进制计算器))（该网站计算的值如果不够8位，需要在高位`左侧`补齐0）

    -   操作 `BitMap` 设置一个字符串

        ```bash
        # 操作 BitMap 使其值为十进制的 1
        127.0.0.1:6379> SETBIT k1 7 1  # 0000 0001 
        (integer) 0
        127.0.0.1:6379> get k1
        "\x01"
        
        # 操作 BitMap 使其值为 a，在 ascii 对应的十进制为 97，用二进制表示为：01100001
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
        
        # 操作 BitMap 的 k2 的值，使 a 更改为 b，
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

        

-   **使用场景**（用户 id 只能是数字类型）

    -   统计任意时间内用户登录次数
        -   SETBIT `user:id` time 1
        -   BITCOUNT `user:id`
    -   统计用户登录在某一时间段内活跃情况
        -   SETBIT `time` user-id 1
        -   BIETOP OR  result time1 time2
        -   BITCOUNT result

-   **SETBIT 和 GETBIT 的底层原理**

    -   `Redis` 的 `SETBIT` 命令存储`BitMap`时，采用的`逆序存储`，该逆序顺序对于用户来讲是无感知的。主要目的是`扩展字节数据时，不再需要移动数据`。

    -   以下内容摘抄自**黄健宏的《Redis 设计与实现》** 章节。

    -   `Redis`使用字符串对象保存位数组。因为字符串对象使用的`SDS`数据结构是二进制安全的。

        -   redisObject.type的值为REDIS_STRING，表示这是一个字符串对象。
        -   sdshdr.len的值为1，表示这个SDS保存了一个一字节长的位数组。
        -   buf数组中的buf[0]字节保存了一字节长的位数组。
        -   buf数组中的buf[1]字节保存了SDS程序自动追加到值的末尾的空字符'\0'。

        ![SDS-Structure](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-structure.jpg?raw=true)

        -   存储一个`字节`的位数组：`0100 1101`，`Redis`在保存数组顺序时，与我们书写顺序时完全相反的。也就是`逆序存储`，在数组中的表示为：`1011 0010`。**注意：数组索引顺序依然是从左到右，不是逆序存储的时候采用了逆序索引，这点在后续会明确解释。**

            

            ![SDS-Structure](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-structure-1.jpg?raw=true)

        -   存储多个`字节`的位数组：`1111 0000 1100 0011 1010 0101`，在 `buf数组中`表示为：`1010 0101 1100 0011 0000 1111`。

            ![SDS-Structure](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-structure-2.jpg?raw=true)

        -   `GETBIT <bitarray> <offset>` 命令实现，复杂度为`O(1)`

            -   1.**byte = offset ÷ 8 并向下取整（byte 值是图22-3的 buf[0]、buf[1]、buf[2]  ）**

                -   byte 值记录了`offset` 指定的二进制位保存在位数组的哪个字节。

            -   2.**bit = ( offset mod 8 ) + 1 **

                -   bit 值记录了`offset`指定的二进制位中字节的第几个位置（**不是索引，是位置**）。

            -   3.根据 `byte` 和 `bit` 值，在位数组 `bitarray` 中定位 `offset` 指定的二进制位，返回该位上的值。

            -   `GETBIT <bitarray> 3`

                ![GETBIT-3](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/GETBIT-3.jpg?raw=true)

            -   `GETBIT <bitarray> 10`

                ![GETBIT-10](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/GETBIT-10.jpg?raw=true)

        -   `SETBIT <bitarray> <offset> <value>` 命令实现，复杂度为`O(1)`

            -   1.**len = ( offset ÷ 8 ) + 1**

                -   len 值记录了保存`offset`指定的二进制位需要多少字节（计算 bug[] ）

            -   2.检查 `bitarray` 键保存的位数组**(sdshdr)**的长度是否小于 len。

                -   2.1 如果小于 len，则需要扩展字节。`sdshdr` 空间预分配策略会额外多分配 `len` 个字节的未使用空间，再加上为保存空字符而额外分配的1字节，及扩展后的字节为：**( len × 2 ) + 1**

            -   3.**byte = offset ÷ 8 并向下取整**

                -   byte 值记录了`offset` 指定的二进制位保存在位数组的哪个字节。

            -   4.**bit = ( offset mod 8 ) + 1**

                -   bit 值记录了`offset`指定的二进制位中字节的第几个位置（**不是索引，是位置**）。

            -   5.根据 **byte** 和 **bit** 值，在 `bitarray` 键保存的位数组中定位`offset`指定的二进制位。

                -   首先将指定的二进制位上当前值保存在 `oldvalue` 变量。
                -   然后将新值 `value` 设置为二进制位的值。

            -   6.返回 `oldvalue` 变量值。

            -   `SETBIT <bitarray> 1 1` 无需扩展字节

                ![SETBIT-1](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/SETBIT-1.jpg?raw=true)

                ![SETBIE-1-1](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/SETBIT-1-1.jpg?raw=true)

            -   `SETBIT <bitarray> 12 1`需扩展字节

                ![SETBIT-12](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/SETBIT-12.jpg?raw=true)

                ![SETBIT-12-1](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/SETBIE-12-1.jpg?raw=true)

            -   **SETBIT 如果采用正常书写顺序保存，在每次扩展buf数组之后，程序都需要将位数组已有的位进行移动，然后才能执行写入操作，这比SETBIT命令目前的实现方式要复杂，并且移位带来的CPU时间消耗也会影响命令的执行速度。对位数组0100 1101执行命令SETBIT ＜bitarray＞ 12 1，将值改为0001 0000 0100 1101的整个过程。如下图。**

                ![SETBIT-ORDER](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/SETBIE-12-3.jpg?raw=true)

            -   **关于`逆序存储` 的疑问及解释，如果根据上文的逆序存储方式进行验证，会出现以下几个疑问。最后的 Redis 源码解释了该问题，通过 bit = 7 - ( bitoffset & 0x7 ) 计算，实际上的 setbitCommand 操作将 0 1 2 3 4 5 6 7 的操作倒转为了 7 6 5 4 3 2 1 0。对于用户来讲，该操作是无感知的，所以当验证逆序存储是，就会出现了下边几个疑问。**

                ![Question-1](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-1.jpg?raw=true)

                ![Question-2](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-2.jpg?raw=true)

                ![Question-3](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/%20bitmap-3.png?raw=true)

                ![Reverse-Code](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/bitmap-reverse.png?raw=true)

###  2. Hash

>   进入Redis 客户端使用  help @hash 查看命令

-   `Redis` 的 `Hash` 是一个 `key-value` 结构。外层的哈希（k-v）用到了`Hashtable`，内存的哈希（value 内的 k-v）则使用了两种数据结构来实现，查看哈希对象的编码命令为：`OBJECT ENCODING <key>`

    -   **ziplist**

        -   当 `Hash` 对象满足两个条件使用 `ziplist`，如果不能满足这两个条件的哈希对象需要使用 `hashtable` 编码，以下两个条件可以通过配置文件修改。
            -   哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节`hash-max-ziplist-value 64`
            -   哈希对象保存的键值对数量小于 512 个`hash-max-ziplist-entries 512`

        ![Ziplist](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/hash-ziplist.jpg?raw=true)

    -   **hashtable**

        ![Hashtable](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/hash-table.jpg?raw=true)

-   常用命令

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

-   应用场景

    -   存储对象缓存

### 3. List

>   进入Redis 客户端使用  help @list 查看命令

-   `Redis` 使用 `quicklist` 的双向链表数据结构实现 List。在链表两段插入数据的复杂度为O(1)，在中间操作性能会很差。

    -   `quicklist` 结构

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

            ![quciklist](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/quicklist.jpg?raw=true)

-   常用命令

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

-   使用场景

    -   LREM 可以实现待办事项列表
    -   PUSH 相关命令可以实现评论列表等
    -   POP 相关的命令可以实现阻塞、非阻塞队列

### 4.Set

>   进入Redis 客户端使用  help @set 查看命令

-   `Redis` 使用 `intset` 编码的整数集合或`hashtable` 编码的集合对象实现`无序` Set。

    -   当 `Set` 内所有元素对象都可以被 `Redis` 解释为整型数值时，且元素数量不超过 `512`  个，使用 `intset` 编码。
    -   当 `Set` 内有元素为 `String ` 类型是，使用 `hashtable` 编码，每个元素对应 `hashtable` 字典的键，值全部设置为 `Null`。

    ![Set](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/set-1.jpg?raw=true)

-   常用命令

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

-   应用场景

    -   SCARD 命令实现`唯一计数器`
    -   SADD、SREM、SISMEMBER 命令实现`点赞`、`喜欢`、`Tags`、`投票`、`社交关系（关注）`
    -   SRANDMEMBER、SPOP 命令实现`抽奖`
    -   SINTER 命令实现社交关系的`共同好友`
    -   SUNION 命令实现社交关系的`推荐关注`

### 5.ZSet(Sorted Set)

>   进入Redis 客户端使用  help @sorted_set 查看命令

-   `Redis` 使用 `ziplist` 或者 `skiplist` 的编码实现 `有序的 Sorted set`。

    -   编码转换

        -   有序集合保存的元素数量小于128个，且有序集合保存的所有元素成员的长度都小于64字节，使用 `ziplist`，当元素数量和长度超过规定则使用 `skiplist`。

    -   `ziplist` 编码的压缩列表底层内部，每个集合元素使用两个紧挨一起的压缩列表节点来保存。第一个节点保存元素的`成员（member）`，第二个元素保存元素的`分值（score）`。分值较小的放置在靠近表头的方向，分值较大的放置在靠近表尾的方向。

        ![ziplist](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/ziplist.jpg?raw=true)

        ![ziplist-1](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/ziplist-1.jpg?raw=true)

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

        ![skiplist](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/skiplist.jpg?raw=true)

-   常用命令

    ```bash
    # 添加一个或多个成员，或更新成员的分数（如果已经存在则更新，不存在添加）
    ZADD key [NX|XX] [CH] [INCR] score member [score member ...]
    # 对给定成员进行指定的增量（increment）
    ZINCRBY key increment member
    # 获取成员数量
    ZCARD key
    # 获取成员分数
    ZSCORE key member
    # 计算成员之间的成员数量（包含该成员本身，正数后查，复数前前查）
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

-   应用场景

    -   ZRANGE 命令集合实现排行榜、权重排名

