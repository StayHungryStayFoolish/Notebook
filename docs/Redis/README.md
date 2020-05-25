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

        -   `GETBIT <bitarray> <offset>` 命令实现

            -   1.**byte = offset ÷ 8 并向下取整（byte 值是图22-3的 buf[0]、buf[1]、buf[2]  ）**

                -   byte 值记录了`offset` 指定的二进制位保存在位数组的哪个字节。

            -   2.**bit = ( offset mod 8 ) + 1 **

                -   bit 值记录了`offset`指定的二进制位中字节的第几个位置（**不是索引，是位置**）。

            -   `GETBIT <bitarray> 3`

                ![GETBIT-3](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/GETBIT-3.jpg?raw=true)

            -   `GETBIT <bitarray> 10`

                ![GETBIT-10](https://github.com/StayHungryStayFoolish/Images-Blog/blob/master/redis/GETBIT-10.jpg?raw=true)

            -   

### 2. Hash

3.  **List**
4.  **Set**
5.  **ZSet(Sorted Set)**
