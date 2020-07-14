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

-   用户进程调用 `recvfrom()` 函数，进入**I/O 阶段 1 准备数据**

    -   请求到达时，一般情况下，数据没准备完成，`用户进程` 原地等待将数据拷贝 kernel 缓冲区。
    
-   `kernel` 数据准备完成，进入 **I/O 阶段 2 拷贝进用户进程缓冲区，并返回**

    ![Block-IO](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Blocking-IO-Page.svg)

#### 2. NonBlocking IO

- da 

- da 

  ![NonBlocking-IO](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/NonBlocking-IO-.svg)

#### 3. IO Multiplexing 

- da 

- 0da

  ![IO-Multiplexing](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/IO-Multiplexing.svg)

##### 3.1 select & poll & epoll & Reactor 

- da 

- da 

- da

  ![epoll](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/epoll.svg)

#### 4. Signal Driven IO

- da 

- da

  ![IO-SignalDriven](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/IO-SignalDriven.svg)

#### 5. Asynchronous IO

- da 

- da 

  ![Asynchronous-IO](https://gitee.com/bonismo/notebook-img/raw/master/img/redis/Asynchronous-IO.svg)
