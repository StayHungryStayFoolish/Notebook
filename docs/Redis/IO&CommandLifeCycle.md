# Redis 单线程及命令处理生命周期

## UNIX Network I/O Model

>   介绍 `Redis` 的多路复用前需要先介绍下 `UNIX` 的 IO 模型，以便更好理解 `Redis` 基于单线程为什么处理速度如此之快。

-   **计算机在处理一次 Network I/O 会涉及两个系统对象**
    1.  一个调用该 I/O 的 process（或者 thread），下图描述用 `Application（用户进程） `来表示。
    2.  一个系统内核 kernel，下图描述用 `Kernel（内核） `来表示。
-   **计算机在处理一次 Network I/O 会涉及数据操作两个阶段**
    1.  等待数据准备
    2.  将数据从内核空间的 buffer 拷贝到用户进程空间的 buffer

>   I/O 模型主要区分在两个系统对象和处理阶段。

### UNIX 的 5 中 I/O 模型

>   **I/O 分为 Synchronous（同步）、Asynchronous（异步）两种模型**

-   **Synchronous Model（同步模型）**
    -   **Blocking IO** 阻塞 IO 
    -   **NonBlocking IO** 非阻塞 IO
    -   **IO Multiplexing** IO 多路复用
    -   **Signal Driven IO** 信号驱动 IO
-   **Asynchronous Model（异步模型）**
    -   **Asynchronous IO** 异步 IO

