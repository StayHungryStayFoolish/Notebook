# Redis 单线程及命令处理生命周期

## UNIX Network I/O Model

>   介绍 `UNIX IO` 模型前，简单介绍下涉及的几个基本概念

-   **基本概念**
  
    -   **CPU 与寻址**
      
        -   `CPU` 与外界交互包含三类总线：**地址总线、数据总线、控制总线**
            -   假设 `CPU` 的`地址总线`由 **32** 根线组成，则可以传 **2<sup>32</sup>** 个地址组合，即 **4GB** 内存。
        -   `CPU` 通过 `地址总线` 的 32 根线使用电流发送 `1(高电平)`、`0(低电平)`来模拟 `bit` ，根据传输的 `bit` 值获取实际物理地址。**所以寻址空间大小由地址总线数量决定。**
        
        ![CPU 寻址](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/CPU&寻址.svg)
        
    -   **存储器**
      
        -   物理存储器
            -   硬件的内存条即物理存储器，CPU 
        -   虚拟存储器
            -   为了保证内存内部运行的操作系统的安全性，需要把操作系统运行使用的这部分内存保护起来，解决该安全问题，需要使用虚拟存储器，系统运行占用独立一块安全内存空间，每一个进程创建单独开辟的内存空间。
        
    -   **内核空间&用户空间**
      
        -   现代操作采用虚拟存储器，Linux 系统会开辟**最高的 1G 字节**（虚拟地址从 `0xC0000000` 到 `0xFFFFFFFF`）供内核使用，成为内核空间。
        -   **较低的 3G 字节**（从虚拟地址 `0x00000000` 到 `0xBFFFFFFF`）供各个进程使用，成为用户空间。
        
    -   **进程切换**
    
        -   为了控制进程的执行，内核必须有能力挂起正在 `CPU` 上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为**进程切换**。任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。
    
    -   **进程阻塞**
    
        -   正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由**系统自动执行阻塞原语（Block）**，使自己由运行状态**变为阻塞状态**。可见，进程的阻塞是进程自身的一种**主动行为**，也因此只有处于运行态的进程（**获得CPU**），才可能将其转为阻塞状态。`当进程进入阻塞状态，是不占用 CPU 资源的`。
    
    -   **文件描述符 fd**
    
        -   `File Descriptor（文件描述符）`是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。
        -   文件描述符在形式上是一个**非负整数**。实际上，它是一个 `索引值`，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于 `UNIX、Linux` 这样的操作系统。
    
    -   **标准 I/O**
    
        -   **标准 I/O 又称缓存 I/O**，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 `UNIX、Linux` 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，`数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间`。
            -   **缺点**
                -   数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

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

#### 1. Blocking IO

> 用户进程从始至终处于 Blocking 状态

- 用户进程发起 `recvfrom()` 函数调用，请求到达时，一般情况下，数据没准备完成，`用户进程` 一直等待直到 `kernel` 准备好数据。

-   `kernel` 数据准备完成，由 `kernel` 向`用户空间` 拷贝数据，完成后向 `用户进程` 返回 ok。

    ![Block-IO](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Blocking-IO-Page.svg)

#### 2. NonBlocking IO

> 用户进程在 kernel 数据准备完成前处于 NonBlocking 状态，在 kernel 数据准备完成后发起 system call 处于 Blocking 状态

- 用户进程发起 `recvfrom()` 函数调用，请求到达时，数据未准备完成，直接返回 `EWOULDBLOCK`。然后用户不断轮询直到 `kernel` 数据准备完成。

- 用户进程再次发起 `recvfrom()` 函数调用，由 `kernel` 向 `用户空间` 拷贝数据，完成后向 `用户进程` 返回 ok。

  ![NonBlocking-IO](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/NonBlocking-IO-.svg)

#### 3. IO Multiplexing 

> 用户进程在执行 `select / poll / epoll` 函数时处于 Blocking 状态，此时用户进程内  `select / poll / epoll`  监听的 socket 处于 `NonBlocking` 状态，让其中一个 Socket 套接字可读（数据准备完成）发起 `recvfrom()` 调用，再次处于 Blocking 状态。

> IO Multiplexing 与 Blocking IO 不同之处在于前者根据 CPU 处理速度比 Network IO 速度快的特点，将多个 IO 请求集中处理，而后者一次只能处理一次 IO 请求。

- 用户进程使用 `select / poll / epoll` 其中一个函数将 `多个 IO 请求(多个客户端连接)` 发起系统调用。然后  `select / poll / epoll` 轮询 `多个 IO 产生的 Socket`，直到其中一个 `Socket 套接字可读` 后返回可读条件。

- 用户进程发起 `recvfrom()` 调用，等待由 `kernel` 向 `用户空间` 拷贝数据，完成后向 `用户进程` 返回 ok。

  ![IO-Multiplexing](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/IO-Multiplexing.svg)

##### 3.1 select & poll & epoll & Reactor 

- **select / poll 存在的问题**

    1.  `select` 监控的 `FD Set` 为 **1024**，`poll` 没有数量限制（但同样存在 2、3 问题）。
    2.  在请求 `kernel` 准备数据前，`select` 和 `poll` 需要先将 `FD Set` 从 `用户空间` 拷贝到 `kernel`。
    3.  当被监控的 `FD Set` 有数据可读时，用户进程需要遍历整个 `FD Set` 来获取已经准备就绪的 `FD`，所以当并发量大时，性能呈线性下降。

- **epoll 解决了上述三个问题**

    - `epoll` 引入了**事件驱动机制，并使用了 Reactor 设计模式**。
      - 通过 `epoll_create` 向内核申请事件队列**（事件队列每个元素由 FD 和事件绑定组成）**
      - 调用低频的 `epoll_ctl` 操作（EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL）**任务队列（Task Queue）**最终添加需要监控的 `FD 及绑定的 Event`。
      - 因为**任务队列**中有不同的事件类型，所以任务队列中的事件最终由**事件分发器（Event Dispatcher）分发到事件处理器（Event Handle），通过事件处理器的不同函数来执行不同的事件。**
      - **当事件完成就绪后**，将可读就绪的 `Event` 递交给高频调用的 `epoll_wait` 队列中。
      - 最终由 `kernel` 向 `用户空间` 拷贝数据。
    - `epoll` 高频的 `epoll_wait` 可读就绪后返回的 `FD Set` 数据拷贝
        - `epoll` 通过 `kernel`与 `用户空间 mmap(内存映射)` 共用同一块内存来解决。`mmap` 将用户空间的一块地址和  `kernel` 空间的一块地址同时映射到相同的一块物理内存地址（不管是用户空间还是内核空间都是虚拟地址，最终要通过地址映射映射到物理地址），使得这块物理内存对内核和对用户均可见，减少用户态和内核态之间的数据交换。

-    **Socket 事件**的 `wakeup callback` 机制

  - **Linux(2.6+ kernel)** 事件 `wakeup callback` 机制，这是 **I/O 多路复用机制存在的本质**。
  - **Linux** 通过 `Socket` 睡眠队列来管理所有等待 `Socket` 的某个事件的 `用户进程`，同时通过 `wakeup` 机制来 `异步唤醒` 整个睡眠队列上等待事件的 `用户进程`，通知 `用户进程` 相关事件发生。通常情况，`Socket` 的事件发生的时候，会**顺序遍历 Socket 睡眠队列上的每个用户进程节点**，调用每个用户进程节点挂载的 `callback` 函数。在遍历的过程中，如果遇到某个节点是排他的，那么就终止遍历，总体上会涉及两大逻辑：
    1.  睡眠等待逻辑
      1.  `select / poll / epoll_wait` 进入内核，判断监控的 `Socket` 是否有关心的事件发生了，如果没，则为当前 `用户进程` 构建一个 `wait_entry` 节点，然后插入到 `监控 Socket 的 sleep_list`。
      2.  进入循环的 `schedule` 直到关心的时间发生了
      3.  关心的时间发生后，将当前 `用户进程` 的 `wait_entry` 节点从 `Scoket` 的 `sleep_list` 中删除
    2.  唤醒逻辑
      1.  `Socket` 的事件发生了，然后 `Socket` 顺序遍历其睡眠队列，依次调用每个`wait_entry` 节点的 `callback函数`。
      2.  直到完成队列的遍历或遇到某个 `wait_entry` 节点是排他的才停止。
      3.  一般情况下 `callback` 包含两个逻辑：1. wait_entry 自定义的私有逻辑、2.唤醒的公共逻辑。主要用于将该 `wait_entry` 的 `用户进程` 放入 `CPU` 的就绪队列，让 `CPU` 随后可以调度其执行。
  
- **epoll 三个重要函数**

- ```c
  int epoll_create(int size)
  int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
    int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
    ```
    
    
  
  ![epoll](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/epoll.svg)

#### 4. Signal Driven IO

>   用户进程在执行 `SIGIO` 函数时处于 Asynchronous 状态，此时用户进程可以执行其他操作，当 `kernel` 递交 `SIGIO` 信号时，用户进程发起系统调用，在此期间出于 Blocking 状态。

- 用户进程简历 `SIGIO`  信号处理程序，通知 `kernel` 需要的数据并直接返回。

- `kernel` 准备数据完成后，向 `用户进程` 递交 `SIGIO` 信号，用户进程收到该信号后发起 `recvfrom()` 系统调用，等待 `kernel` 向 `用户空间` 拷贝数据，完成后向 `用户进程` 返回 ok。

  ![IO-SignalDriven](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/IO-SignalDriven.svg)

#### 5. Asynchronous IO

>   用户进程发起系统调用后处于 Asynchronous 状态，当 `kernel` 递交执行信号时，用户进程处理 `kernel` 拷贝数据到用户空间阶段处于  Blocking 状态**（因为此时是 Write 状态）**。

- 用户进程发起 `aio_read` 通知 `kernel` 需要的数据，然后用户进程可以执行其他操作。

- `kernel` 准备好数据后，通知 `用户空间` 数据准备完成，在 `aio_read` 中递交指定的信号

-  `用户进程` 处理 `kernel` 拷贝数据到 `用户空间`。**此阶段 Blocking 状态**

  ![Asynchronous-IO](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Asynchronous-IO.svg)

## Redis 命令周期

>   服务器处理客户端命令的流程为：1.服务器启动监听；2. 接收命令请求并解析；3. 执行命令请求；4. 返回命令回复 

## Redis 事件

>   Redis 服务器是典型的事件驱动程序，Redis 将事件分为两大类：文件事件和时间事件（Redis 内只有一种时间事件）。

-   **文件事件**
    -   Socket 的读写时间
-   **时间事件**
    -   处理一些周期性执行的定时任务
