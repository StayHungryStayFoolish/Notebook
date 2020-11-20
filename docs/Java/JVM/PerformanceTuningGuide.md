# JVM 性能调优指南
**Java 应用性能优化依次可以划分为 4 个层级**

1.  **应用层**
    -   优化代码结构，避免线程阻塞、死锁等情况
2.  **数据库层**
    -   优化 SQL 语句，避免数据库全表扫描、死锁等情况
3.  **架构层**
    -   在理解架构机制的情况下进行最优配置，提高计算、存储能力，降低系统负载
4.  **JVM 层**
    -   通过对应用程序的压测，不断调整 JVM 参数，最终达到低延迟、高吞吐量、最优内存使用率三个性能指标

Java 应用程序影响性能的因素非常多，比如磁盘、内存、IO等因素。以上 4 个层级优化难度是依次递增的，架构层优化对系统性能影响最大。

**本文假定前三者已经优化之后，仍需提高性能，那么 JVM 则是最后一次提升性能的关键操作，调整的最终目标是以最低的硬件消耗成本使应用程序具有更高的性能。**

## 1. JVM 性能监测工具

### Application Performance Management(APM)

#### APM 与 Metrics 两个体系的区别

严格意义上，目前只有 APM 的定义，所以把偏硬件监测的暂定为 Metrics。

两个体系都属于监控系统，但却有很大的区别，因为本文涉及 JVM 性能监测，所以此处只做简单记录。

**APM** 体系都是基于 [Google Dapper](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf) 的一篇论文实现的，旨在**监测程序内部执行过程和服务间链路调用情况，更利于深入剖析关于代码层面的问题，侧重于 Tracing 方向。**

-   APM 体系开源项目
    1.  [Pinpoint](https://github.com/pinpoint-apm/pinpoint)
    2.  [SkyWalking](https://github.com/apache/skywalking)
    3.  [Glowroot](https://github.com/glowroot/glowroot)
    4.  [Zipkin](https://github.com/openzipkin/zipkin)
    5.  [Kamon](https://github.com/kamon-io/Kamon)
    6.  [Stagemonitor](https://github.com/stagemonitor/stagemonitor)
    7.  [CAT](https://github.com/dianping/cat)

推荐使用 `Pinpoint` 或者 `SkyWalking`。因为这两个都是通过字节码注入，对代码完全无侵入。并且 `SkyWalking` 是国内大神**吴晟**开源的，目前已在 `Apache` 孵化，且社区活跃。`Pinpoint` 是韩国公司开源，社区活跃度与沟通上相对差一些。具体性能对比可自行了解。`Glowroot` 对代码也是无侵入的，`Kamon` 需要在程序入口加入少量代码，并在 JVM 以参数插件启动，`Stagemonitor` 和 `Kamon` 使用方式类似。 `Zipkin` 经常与 `Spring Cloud Slueth` 组合，完成全链路监控的作用，并且 `Zipkin` 还可以和 `ELK` 组合来记录日志和展示。`CAT` 是大众点评的开源项目，侵入性较大。

**Metrics** 体系旨在**监控服务器硬件指标与系统服务运行状态等**。其中 `Prometheus` 基于 `Google Borgmon` 内部原理设计而来。

-   Metrics 体系开源项目
    1.  [Prometheus](https://github.com/prometheus/prometheus)
    2.  [Zabbix](https://github.com/zabbix/zabbix)

`Prometheus` 通常和 `Grafana` 配合使用，因为 `Grafana` 只负责绘图，并且 UI 比较酷炫。

## 2. JDK 自带工具

上边介绍了一些功能强大的监测工具，都包含了 JVM 性能监测。在实际生产部署过程中，可能会选择适合的工具。但是如果只想关注 JVM ，则可是使用以下功能相对单一的工具。

`Mac OS` 终端输入`/usr/libexec/java_home -V` 进入 `bin` 路径可以看到 JDK 自带的这些工具。

-   **Monitoring Tools**
    -   **jps** - 查询 JVM 运行进程状态信息，类似 ps 命令子集。
    -   **jstat** - 实时本地、远程监控 JVM 内存、类加载、垃圾收集、JIT 编译等，但是当显示 GC 信息时，仅在 GC 之后会更新最新值，这点不像 **jcmd** 永远显示实时信息。  
    -   **jstatd** - 基于 **RMI 协议**的服务程序，相当于 JVM 的代理服务器，JDK 很多自带的工具都需要基于 `jstatd` 进行远程连接，用于监控 JVM 状态，需要在服务器机器设置一个后缀为 `.policy` 的安全策略文件。
-   **Troubleshooting Tools**
    -   **jinfo** - 查看 JVM 配置参数，还可以动态设置部分参数值。`jinfo` 使用时需要 `attach` 到目标 JVM 上，**JDK 8 存在严重 bug，会导致进程结束，在 JDK 11 之后才修复**。
    -   **jhat** - 用于解析堆、支持生成堆信息文件，并对转储文件进行展示分析，服务启动后，可以通过 http://127.0.0.1:7000/ 查看，**配合 jmap 生成堆转储文件，然后解析**。
    -   **jmap** - 打印 JVM 堆概要信息，包括堆配置、新生代、老生代信息，支持**导出文件**，并结合 `jvisualvm`、`MAT`、`JProfile` 等工具进行分析。当使用 `jamp -histo:live pid` 携带 `:live` 命令只打印活动类信息，会触发 GC，导致 `STW`，生产环境慎用，**JDK 8 存在严重 bug，会导致进程结束，在 JDK 11 之后才修复**。
    -   **jstack** - 查看 JVM 进程内当前时刻的线程快照，定位线程停顿、死锁等长时间等待的问题，支持导出线程相关信息。
-   **Troubleshooting, Profiling, Monitoring and Management Tools**
    -   **jcmd**- 用于将诊断命令请求发送到 JVM，不支持远程连接 JVM。 
    -   **jconsole** - 实时本地、远程监控 CPU、JVM 内存、类加载、垃圾收集、线程（检测死锁）和类等的监控，是一个基于 JMX（java management extensions）的 GUI 性能监测工具。
    -   **jvisualvm** - `jmap、jinfo、jstat、jstack、jconsole` 功能合集，支持多种功能插件，因为 `jvisualvm` 不仅支持 JMX 还支持 Jvmstat、Attach API 和 SA 等监测技术，还可以记录有关 JVM 实例的数据，并将该数据保存到本地系统，便于分析，支持**内存泄漏检测**。
    -   **jmc** - `JDK Mission Control` 相对 `jconsole`、`jvisualvm` 可以展示更多的信息，也是 GUI 监测工具，UI 更美观，在 `JDK 8u261` 被移除，[Oracle JDK 8u261更新文档](https://www.oracle.com/java/technologies/javase/8u261-relnotes.html)，可在 [Oracle JMC下载](https://www.oracle.com/cn/javase/jmc/) 并参考 [Oracle JMC 安装](https://www.oracle.com/java/technologies/javase/jmc-install.html)。

以上基本介绍来自 [Oracle Tools 文档](https://docs.oracle.com/javase/8/docs/technotes/tools/)

**GC 在线解析工具：** [GCeasy](https://gceasy.io/)

### 2.1 JVM 两种通信方式

#### 2.1.1 Java Remote Method Invocation(Java RMI)

**[Wiki 关于 RMI 的解释：](https://en.wikipedia.org/wiki/Java_remote_method_invocation)** 在计算机中，Java 远程方法调用（Java RMI）是一种执行远程方法调用的 Java API，相当于面向对象的远程过程调用（RPC），支持直接传输序列化的 Java 类和分布式垃圾回收。

##### RMI 配置

**远端服务器**

1.  查询 $JAVA_HOME Path

```shell
echo $JAVA_HOME
/usr/local/jdk1.8.0_201
```

2.  创建安全策略 `.policy`  的文件，并记录 `.policy` 文件路径

```shell
cd /home
mkdir admin
vim jstatd.java.policy

// 此处指定 $JAVA_HOME 路径
grant codebase "file:/usr/local/jdk1.8.0_201/lib/tools.jar"{
    permission java.security.AllPermission;
};
```

3.  启动 `jstatd` 服务，在 `-J-Djava.security.policy` 参数指定 `.policy` 文件路径，以监测 JVM，**实际输入参数需要删除注释行**

```shell
jstatd \
# 指定安全策略文件
-J-Djava.security.policy=/home/admin/jstatd.java.policy \
# 是否打印调用日志
-J-Djava.rmi.server.logCalls=true \
# 指定 hostname
-J-Djava.rmi.server.hostname=***.**.**.*** \
# 指定端口（默认端口 1099），并后台启动
-p *** &
```

**本地连接**

**jps 连接远端 `jps -outputOptions rmi://hostname:port`**

**jstat 连接远端 `jstat -outputOptions vmid@hostname`**

**异常处理**

#### 2.1.2 Java Management Extensions(JMX)

**[Wiki 关于 JMX 的解释](https://en.wikipedia.org/wiki/Java_Management_Extensions)：**Java 管理扩展（JMX）是一种 Java 技术，它为管理和监控应用程序、系统对象、设备（如打印机）和面向服务的网络提供工具。这些资源由称为 **MBeans（Managed Bean的缩写）**的对象来表示。在 API 中，类可以被动态加载和实例化。管理和监控应用程序可以使用 `Java Dynamic Management Kit(Java动态管理工具包)`  来设计和开发。

**JMX 使用的 RMI 协议进行通信，被称为 Java Remote Message Protoco(JRMP)，由 `javax.management.remote.rmi.RMIConnector` JVM 中的实现，该协议要求服务端和客户端都需要用 Java 编写。**

##### JMX 配置

**参考 JVisualvm 的 JMX 配置**

### 2.2 jps tools

[Oracle jps command](https://docs.oracle.com/en/java/javase/13/docs/specs/man/jps.html)

| Jps 命令            | 显示结果                                   |
| ------------------- | ------------------------------------------ |
| jps                 | 显示 vmid、启动类名称                      |
| jps -m              | 显示 vmid、传递 main 方法的参数            |
| jps -l              | 显示 vmid、启动类完成包名或 jar 完整路径名 |
| jps -v              | 显示 vmid、传递给 jvm 参数                 |
| jps -V              | 显示 vmid、启动类名、jar 文件名            |
| m、l、v、V 任意组合 | 显示结果组合                               |

### 2.3 jstat tools

[Oracle jstat command](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)

**jstat 命令格式**

```shell
jstat -outputOptions [ -t] [-hlines] vmid [interval [count] ]

参数描述：
-t:	显示一个时间戳列作为输出的第一列。时间戳是自目标JVM的启动时间以来的时间。vmid 	 
-h n:	每 `n` 个输出行显示一个列标题，其中`n`是一个正整数。默认值为`0`，它显示数据第一行的列标题。
vmid:	当前 JVM 线程 id
interval:	指定单位的采样间隔，秒（s）或毫秒（ms）。默认单位是毫秒。这必须是一个正整数。当指定时，jstat命令会在每个时间间隔产生输出。
count:	要显示的样本数量。默认值是无穷大，这会导致jstat命令显示统计数据，直到目标JVM终止或jstat命令终止。这个值必须是一个正整数。
```

**jstat Options**

| 命令              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| -class            | 显示有关类加载器行为的统计信息                               |
| -compiler         | 显示有关 `HotSpot VM` 即时编译器行为的统计信息               |
| -gc               | 显示有关垃圾收集堆行为的统计信息                             |
| -gccapacity       | 显示内存所有分代及其相应空间的统计数据                       |
| -gccause          | 显示有关垃圾收集统计信息的摘要（`-gcutil` 与相同），以及最近和当前（如果适用）垃圾收集事件的原因 |
| -gcnew            | 显示有关 `Young Gen` 行为的统计信息                          |
| -gcnewcapacity    | 显示有关 `Young Gen` 大小及其相应空间的统计信息              |
| -gcold            | 显示有关 `Old Gen` 行为的统计信息和元空间统计信息            |
| -gcoldcapacity    | 显示有关 `Old Gen` 的大小的统计信息                          |
| -gcmetacapacity   | 显示有关 `Metaspace` 大小的统计信息                          |
| -gcutil           | 显示有关  GC 收集统计信息的摘要                              |
| -printcompilation | 显示 `HotSpot VM` 编译方法统计信息                           |

**jstat 显示结果参数描述**

| 参数描述 | jstat 命令                                                   | jps 命令                                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| S0C      | 以字节为单位显示 **Survivor 0** 区域的当前大小               | -gc<br/>-gccapacity<br/>-gcnew<br/>-gcnewcapacity            |
| S1C      | 以字节为单位显示 **Survivor 1** 区域的当前大小               | -gc<br/>-gccapacity<br/>-gcnew<br/>-gcnewcapacity            |
| S0U      | 以字节为单位显示 **Survivor 0** 区域的当前使用情况           | -gc<br/>-gcnew                                               |
| S1U      | 以字节为单位显示 **Survivor 1** 区域的当前使用情况           | -gc<br/>-gcnew                                               |
| EC       | 显示 `Eden` 的当前大小，以 KB 为单位                         | -gc<br/>-gccapacity<br/>-gcnew<br/>-gcnewcapacity            |
| EU       | 显示 `Eden Space` 的当前使用情况，以 KB 为单位               | -gc<br/>-gcnew                                               |
| OC       | 显示 `Old Gen` 的当前大小，以 KB 为单位                      | -gc<br/>-gccapacity<br/>-gcold<br/>-gcoldcapacity            |
| OU       | 显示 `Old Gen` 的当前使用情况，以 KB 为单位                  | -gc<br/>-gcold                                               |
| MC       | 显示 `Metaspace` 的当前大小，以 KB 为单位                    | -gc<br/>-gccapacity<br/>-gcold<br/>-gcoldcapacity<br/>-gcmetacapacity |
| MU       | 显示 `Metaspace` 的当前使用情况，以 KB 为单位                | -gc<br/>-gcold                                               |
| MCMN     | `Metapsace` 初始化大小，以 KB 为单位                         | -gccapacity<br/>-gcmetacapacity                              |
| MCMX     | `Metapsace` 最大容量，以 KB 为单位                           | -gccapacity<br/>-gcmetacapacity                              |
| YGC      | `Young Gen` 发生 GC 事件的次数                               | -gc<br/>-gccapacity<br/>-gcnew<br/>-gcnewcapacity<br/>-gcold<br/>-gcoldcapacity<br/>-gcmetacapacity<br/>-gcutil<br/>-gccause |
| YGCT     | `Young Gen` 的 GC 累计时间                                   | -gc<br/>-gcnew<br/>-gcutil<br/>-gccause                      |
| FGC      | `Full GC` 发生 GC 事件的次数                                 | -gc<br/>-gccapacity<br/>-gcnewcapacity<br/>-gcold<br/>-gcoldcapacity<br/>-gcmetacapacity<br/>-gcutil<br/>-gccause |
| FGCT     | `Full GC` 累计时间                                           | -gc<br/>-gcold<br/>-gcoldcapacity<br/>-gcmetacapacity<br/>-gcutil<br/>-gccause |
| GCT      | GC 操作累积的总时间                                          | -gc<br/>-gcold<br/>-gcoldcapacity<br/>-gcmetacapacity<br/>-gcutil<br/>-gccause |
| NGCMN    | `Young Gen` 初始化大小，以 KB 为单位                         | -gccapacity<br/>-gcnewcapacity                               |
| NGCMX    | `Young Gen` 最大容量，以 KB 为单位                           | -gccapacity<br/>-gcnewcapacity                               |
| NGC      | `Young Gen` 当前容量，以 KB 为单位                           | -gccapacity<br/>-gcoldcapacity                               |
| OGCMN    | `Old Gen` 初始化大小，以 KB 为单位                           | -gccapacity<br/>-gcoldcapacity                               |
| OGCMX    | `Old Gen` 最大容量，以 KB 为单位                             | -gccapacity<br/>-gcoldcapacity                               |
| OGC      | `Old Gen` 当前容量，以 KB 为单位                             | -gccapacity<br/>-gcoldcapacity                               |
| CSS      | `Compressed Class Space` 利用率百分比。                      | -gcutil<br />-gccause                                        |
| CCSMN    | `Compressed Class Space` 初始化大小，以 KB 为单位            | -gccapacity<br/>-gcmetacapacity                              |
| CCSMX    | `Compressed Class Space` 最大容量，以 KB 为单位              | -gccapacity<br/>-gcmetacapacity                              |
| CCSC     | `Compressed Class Space` 当前容量，以 KB 为单位              | -gc<br />-gccapacity<br/>-gcold<br />-gcmetacapacity         |
| CCSU     | `Compressed Class Space` 使用容量，以 KB 为单位              | -gc<br />-gcold                                              |
| LGCC     | 上次 GC 的原因                                               | -gccause                                                     |
| GCC      | 当前 GC 的原因                                               | -gccause                                                     |
| TT       | `Young Gen -> Old Gen` 的 `Tenuring Threshold`阈值。<br />如果在年轻区域复制这么多次(S0 ->S1, S1->S0)，<br />它们就会被移动到老区域。 | -gcnew                                                       |
| MTT      | `Young Gen -> Old Gen` 最大阈值                              | -gcnew                                                       |
| DSS      | 所需的 `Survivor Space` 的大小，以 KB 为单位，<br />当 `S0U` 或者 `S1U` 超过 `DSS` 将使当前 `Survivor` 区域对象移至 `Old Gen`，<br />最终会触发 GC ，`YCG` 增加 **1** 次，`YGCT` 时间会变化。 | -gcnew                                                       |

`Compressed Class Space(压缩类空间)` 是 `Hotspot` 在 **64** 位系统的压缩技术，旨在使用 32 位指针找到 64 位地址（通过存储的一个 base 值加上 32 位基准地址值）。

### 2.4 jmap & jhat Tools

```shell
# 查询 pid
jps -l
# 根据 pid 生成堆转储文件 filename.hprof
jmap -dump:format=b,file=filename.hprof pid
# 使用 jhat 解析 filename.hprof 文件，访问 http://localhost:7000
jhat -J-Xmx1024m filename.hprof
# 可以使用 OQL 语言查询
select s from java.lang.String s where s.count > 100
```

[jhat OQL 文档](http://cr.openjdk.java.net/~sundar/8022483/webrev.01/raw_files/new/src/share/classes/com/sun/tools/hat/resources/oqlhelp.html)

### 2.5 jstack Tools

```shell
# 打印 pid 线程信息
jstack pid	
```

| 线程参数                  | 线程状态                                         |
| ------------------------- | ------------------------------------------------ |
| runnable                  | 线程处于运行中                                   |
| deadlock                  | **`死锁`**                                       |
| blocked                   | **`线程阻塞`**                                   |
| Parked                    | 停止                                             |
| locked                    | 对象加锁                                         |
| watiting                  | 线程正在等待                                     |
| waiting to lock           | 等待上锁                                         |
| Object.wait()             | 对象等待中                                       |
| waiting for monitor entry | **`等待获取监视器`**                             |
| Waiting on condition      | **`待资源，最常见的情况是线程在等待网络的读写`** |

### 2.6 jconsole & JVisualvm & JMC Tools

[Oracle 远程链接文档](https://docs.oracle.com/javase/1.5.0/docs/guide/management/agent.html#PasswordAccessFiles)

#### 2.6.1 JMX 远程无认证连接

1.  服务器启动 jar，**实际输入参数需要删除注释行**

```shell
java \
# 服务器地址
-Djava.rmi.server.hostname=***.***.**.** \
# 开启远程链接
-Dcom.sun.management.jmxremote=true \
# jmx 端口
-Dcom.sun.management.jmxremote.port=*** \
# 是否开启认证
-Dcom.sun.management.jmxremote.authenticate=false \
# 是否开启 ssl
-Dcom.sun.management.jmxremote.ssl=false \
-jar ***.jar
```

2. **本地机器**运行 `jvisualvm` 启动，然后添加**远程主机 IP**，配置端口号，远程服务器连接成功。

#### 2.6.2 JMX 远程安全认证连接

1.  **服务器**进入路径 `JRE_HOME/lib/management/`
2.  拷贝 `jmxremote.password.template` 为 `jmxremote.password` 并打开 `monitorRole` 和 `controlRole` 注释，`monitorRole` 只可以监控，`controlRole` 可以在本地服务器进行一些操作，比如堆转储，GC 等操作。

```shell
# The "monitorRole" role has password "QED".
# The "controlRole" role has password "R&D".
monitorRole QED
controlRole R&D
```

3.  使用 JVM 参数启动 `***.jar`，**实际输入参数需要删除注释行**

```shell
java \
# 服务器地址
-Djava.rmi.server.hostname=***.***.**.** \
# 开启远程链接
-Dcom.sun.management.jmxremote=true \
# jmx 端口
-Dcom.sun.management.jmxremote.port=*** \
# 是否开启 ssl
-Dcom.sun.management.jmxremote.ssl=false \
# 是否开启密码认证
-Dcom.sun.management.jmxremote.password=true \
# 是否开启认证，此参数可以省略
-Dcom.sun.management.jmxremote.authenticate=true \
-jar ***.jar
```

4.  **本地机器**运行 `jvisualvm` 启动，然后添加**远程主机 IP**，配置服务器地址 `***.***.**.**`，添加 JMX 连接，配置端口号，输入用户名 `monitorRole` 或 `controlRole`，再输入对应的密码，远程服务器连接成功。
5.  如需开启 SSL 请查阅官方文档。

**以上步骤 1、2、3 属于 JMX 配置，完成后，可以使用支持 JMX 的工具进行连接。**

## 3.  JVM 调优原则

上边介绍了各种 JVM 监测工具的相关配置和部分参数。接下来将介绍如何对 JVM 进行性能调优。

调整的最终目标是以最低的硬件消耗成本使应用程序具有更大的吞吐量。JVM 调整也不例外。

JVM 调优主要涉及优化 **GC(垃圾收集器)** 以获得更好的收集性能，以便在 JVM 上运行的应用程序可以具有更大的吞吐量，同时使用更少的内存并具有更低的延迟。

### 3.1 性能指标

1. **Latency(延迟)**
    -   `GC` 过程中的 `STW` 时间越少，延迟越低。
2. **Throughput(吞吐量)**
    -   应用程序的线程用时与程序总用时的占比。应用程序线程占比越高，吞吐量越高。**99%** 的吞吐量意味着应用程序运行了 **99 秒**，而 `GC` 运行了 **1 秒**。
3. **Memory usage(内存利用率)**
    -   适当的内存能保证一个稳定的内存使用率，可以保证 `GC` 不会因为要回收的垃圾太多而造成较长时间的 `STW`，也可以一定程度避免 `OOM` 问题。

这三个维度中任何一个的性能提升几乎是以其他一个或两个属性的性能损失为代价的。应用程序业务需求确定一个或两个属性对应用程序的重要性。

**延迟与吞吐量是竞争关系**。在 [GC 收集器](http://notebook.bonismo.ink/#/Java/JVM/GarbageCollection) 中提到过 `Application pause` 和 `Application throughput` 两个概念就是本文的延迟与吞吐量。JVM 中有独立的线程（GC Threads）来执行 GC，所以只要 `GC threads` 是活动的就肯定会与 `Application threads` 竞争 CPU 的时钟周期。

因此高吞吐量势必造成 `Heap Memory` 中的对象数量持续攀高，当 `GC` 需要确定 `Heap Memory` 中对象是否可以回收时，需要更长的 `STW` 时间，以便于完成工作。**通常来讲，会在延迟与吞吐量取舍上采取折中方案，推荐采用低延迟、高吞吐量、适当的内存使用率来进行调优。**

### 3.2 性能调整原则

1.  **Minor GC**
    -   每次 `Minor GC` 尽可能多的收集垃圾对象，减少 `Full GC` 频率。
2.  **GC 内存最优原则**
    -   选择最优内存，以达到低延迟和高吞吐量的性能。内存并不是越大越好，内存越大，`GC ` 回收造成的 `STW` 时间就越长，相反内存越小，`GC` 的频率就越高。
3.  **GC 三分之二原则**
    -   按照应用程序需求在**三个性能指标**选择最优化。

## 4. 调优过程

![Tuning-JVM](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/jvm-tuning-process.png)


**JVM 调整涉及连续的配置优化和基于性能测试结果的多次迭代。**在满足每个所需的系统指标之前，每个先前的步骤可能会经历多次迭代。在某些情况下，为了满足特定指标，可能需要多次调整先前的参数，从而需要再次测试所有先前的步骤。

此外，调整通常从满足应用程序的`内存使用`要求开始，然后是`延迟时间`和`吞吐量`。调整应遵循以下步骤顺序。

### 4.1 JVM 调优参数表

>   -XX:+	 `+` 代表开启
>
>   -XX:-	  `-` 代表关闭
>
>   -XX:=	  `=` 代表指定一个值、文件路径、指令等
>
>   **JVM 配置数值只支持整数类型，不支持带小数的值。**

[Oracle JVM 8 官方文档参数](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)

[Oracle JVM 9 官方文档参数](https://docs.oracle.com/javase/9/tools/java.htm#JSWOR624)

[JVM 参数列表](http://pingtimeout.github.io/jvm-options/#)

**强烈推荐：**[JVM 在线设置参数工具](http://jvmmemory.com/)

| 参数                                               | JVM 9 ~ N 参数                                     | 默认值 / 格式                                                | 说明                                                         |
| :------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **系统设置**                                       |                                                    |                                                              |                                                              |
| `-Duser.timezone=US/Eastern`                       | `-Duser.timezone=US/Eastern`                       |                                                              | 如果 Java 代码中有使用 `java.util.Date` 和 `java.util.Calendar` 则可以使用该参数**指定时区** |
| **-d64<br />-d32**                                 | **-d64<br />-d32**                                 |                                                              | 如果OS环境中同时安装了 **32位** 和 **64位** 软件包，JVM 会自动选择32位环境软件包 |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 垃圾收集器**                               | **JVM 9 ~ N 垃圾收集器**                           | **JVM  垃圾收集器**                                          | **JVM 垃圾收集器**                                           |
| -XX:UseSerialGC                                    | -XX:UseSerialGC                                    | `响应速度优先`                                               | 串行GC                                                       |
| -XX:+UseParallelGC                                 | -XX:+UseParallelGC                                 | `吞吐量优先`                                                 | 并行 GC，`Young Gen` 多线程收集，`Old Gen` 单线程收集        |
| -XX:+UseParallerOldGC                              | -XX:+UseParallerOldGC                              | `吞吐量优先`                                                 | 并行 GC，`Young Gen` 和 `Old Gen` 都是多线程收集             |
| -XX:+UseConcMarkSweepGC                            | -XX:+UseConcMarkSweepGC                            | `响应速度优先`                                               | 并发收集                                                     |
| -XX:UseG1GC                                        | -XX:UseG1GC                                        | `响应速度优先`                                               | CMS 替代版本                                                 |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 远程连接设置**                             | **JVM 9 ~ N 远程连接设置**                         | **JVM 远程连接设置**                                         | **JVM 远程连接说明**                                         |
| **-Dsun.net.client.defaultConnectTimeout=2000**    | **-Dsun.net.client.defaultConnectTimeout=2000**    | **单位 ms，默认值 -1，表示不设置超时。**                     | 多种协议（即SOAP，REST，HTTP，HTTPS，JDBC，RMI等）与远程应用程序连接，有时可能需要很长才能响应或者不响应。因此可以设置该参数。该选项设置`主机连接超时` |
| **-Dsun.net.client.defaultReadTimeout=2000**       | **-Dsun.net.client.defaultReadTimeout=2000**       | **单位 ms，默认值 -1，表示不设置超时。**                     | 该选项设置`建立资源连接超时`                                 |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 内存配置**                                 | **JVM 9 ~ N 内存配置**                             | **JVM 内存配置格式**                                         | **JVM 内存配置说明**                                         |
| **JVM 8 堆外内存设置**                             |                                                    |                                                              |                                                              |
| -XX:MaxDirectMemorySize=1G                         | -XX:MaxDirectMemorySize=1G                         |                                                              | 当 IO 操作时可能会遇到 `java.lang.OutOfMemoryError: Direct buffer memory` 此时需要调整堆外内存大小以避免类似问题 |
| **设置 `Stack Memory` 空间**                       |                                                    |                                                              |                                                              |
| `-Xss1024k`                                        | `-Xss1024k`                                        | **默认值 1M**                                                | `Stack Memory` 每个线程栈的大小，用来保存当前线程局部变量、返回值、对象指针等，当使用空间超过指定值，会抛出 `StackOverflowError`。 |
| **设置 `Heap Memory` 空间**                        |                                                    |                                                              |                                                              |
| -Xmx2048m                                          | -Xmx                                               |                                                              | 最大 `Heap Memory`                                           |
| -Xms2048m                                          | -Xms                                               |                                                              | 初始 `Heap Memory`，一般和 `-Xmx` 一样，不留 `reserved`      |
| **设置 `Young Gen` 空间**                          |                                                    |                                                              |                                                              |
| -Xmn800m                                           | -Xmn                                               |                                                              | `Young Gen` 大小                                             |
| -XX:MaxTenuringThreshold=15                        | -XX:MaxTenuringThreshold=15                        | **默认值 15**                                                | `Young Gen` 晋升到 `Old Gen` 的阈值                          |
| -XX:NewRatio=2                                     | -XX:NewRatio=2                                     | **默认值 1:2**                                               | `Young Gen`:`Old Gen` = 1:2                                  |
| -XX:SurvivorRatio=8                                | -XX:SurvivorRatio=8                                | **默认值 8:1:1**                                             | `Eden`:`Survivor`:`Survivor` = 8:1:1                         |
| -XX:+UseTLAB                                       | -XX:+UseTLAB                                       | **默认启用**                                                 | 在 `Young Gen` 分配 `TLAB` 所需的内存空间                    |
| `-XX:TLABSize=512k`                                | `-XX:TLABSize=512k`                                | **默认值 512 KB**                                            | 设置 `TLAB` 大小                                             |
| **设置 `Metaspace` 空间(可以理解成 Method Area)**  |                                                    |                                                              |                                                              |
| -XX:MetaSpaceSize=50m                              | -XX:MetaSpaceSize=                                 | **默认值 20 MB，根据 OS 变化**                               | 每次触发 `Full GC` 扩容的阈值                                |
| -XX:MaxMetaSpaceSize=2048m                         | -XX:MaxMetaSpaceSize=                              | **默认值为 OS 内存**                                         | `Metaspace` 最大值，建议设置最大值，因为默认是系统内存值，容易导致系统内存不够。 |
| -XX:MinMetaspaceFreeRatio=40                       | -XX:MinMetaspaceFreeRatio=40                       |                                                              | `Full GC` 后 `Metaspace` 剩余空间**小于**该参数则**扩容**    |
| -XX:MaxMetaspaceFreeRatio=90                       | -XX:MaxMetaspaceFreeRatio=90                       |                                                              | `Full GC` 后 `Metaspace` 剩余空间**大于**该参数则**缩容**    |
|                                                    |                                                    |                                                              |                                                              |
| **`JVM 优化（吞吐量、延迟、JIT）`**                | **`JVM 优化（吞吐量、延迟、JIT）`**                | **`JVM 优化（吞吐量、延迟、JIT）`**                          | **`JVM 优化（吞吐量、延迟、JIT）`**                          |
| `-XX:MaxGCMinorPauseMillis=1`                      | `-XX:MaxGCMinorPauseMillis=1`                      | **默认单位值毫秒** `优化延迟`                                | 设置 MinorGC 最大暂停时间，**软目标，JVM 会尽力实现          |
| `-XX:MaxGCPauseMillis=500`                         | `-XX:MaxGCPauseMillis=500`                         | **默认单位值毫秒** `优化延迟`                                | 设置GC 最大暂停时间，**软目标，JVM 会尽力实现**              |
| `-XX:GCTimeRatio=nnn`                              | `-XX:GCTimeRatio=nnn`                              | **`提高吞吐量，默认值 99%` 假设值为 nnn，则比例为 1 / (1 + nnn)，该值是 GC 的占比** | **应用程序线程的执行时间 / 总程序执行时间 = 目标比值，100% - 99% = 1% 意味着 GC 时间占比为 1%**，具体可以参考 [Oracale Ergonomics 文档](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gc-ergonomics.html) |
| `-XX:+UseAdaptiveSizePolicy`                       | `-XX:+UseAdaptiveSizePolicy`                       | **默认启动，此参数和 `-XX:SurvivorRatio=8` 冲突**            | 自动选择 `Eden Gen` 和 `Survivor` 比例                       |
| `-XX:+AggressiveOpts`                              | `-XX:+AggressiveOpts`                              |                                                              | 启用积极的性能优化功能                                       |
| -XX:+TieredCompilation                             | -XX:+TieredCompilation                             |                                                              | 开启分层编译，开销小                                         |
| `-XX:CompileThreshold=1500`                        | `-XX:CompileThreshold=1500`                        | **默认值 1500**                                              | 设置 `JIT` 编译次数，如果启用分层编译，忽略此参数            |
| -XX:+UseCompressedOops                             | -XX:+UseCompressedOops                             | **默认开启**                                                 | 开启 **32位** 表示 **64位** 的普通对象指针，**64 位系统有效** |
| -XX:+UseCompressedClassPointers                    | -XX:+UseCompressedClassPointers                    | **默认开启**                                                 | 开启类压缩。用 **32位** 表示 **64位** 类指针，**64 位系统有效** |
| -XX:CompressedClassSpaceSize=1G                    | -XX:CompressedClassSpaceSize=1G                    | **默认值 1G**                                                | 设置类压缩空间大小                                           |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 打印日志**                                 | **JVM 9 ~ N 打印日志**                             | **JVM 打印日志格式**                                         | **JVM 打印日志说明**                                         |
| **JVM 8 日期**                                     | **JVM 9 ~ N 日期参数**                             | **JVM 日期**                                                 | **JVM 日期说明**                                             |
| `-XX:+PrintGCDateStamps`                           | -Xlog:gc*::timemillis                              | 2013-05-04T 21:53:59.234+0800<br />JVM 9 显示时间戳`(System.currentTimeMillis())` | 当前时间                                                     |
| `-XX:+PrintGCTimeStamps`                           | -Xlog:gc*::time                                    | 0.004s                                                       | JVM 启动时间                                                 |
| `-Dsun.rmi.dgc.client.gcInterval=3600000`          | `-Dsun.rmi.dgc.client.gcInterval=3600000`          | **RMI 监控配置 Full GC 显式收集频率**                        | **如果使用 RMI 监控，默认 1小时显式进行 Full GC**            |
| `-Dsun.rmi.dgc.server.gcInterval=3600000`          | `-Dsun.rmi.dgc.server.gcInterval=3600000`          | **RMI 监控配置 Full GC 显式收集频率**                        | **如果使用 RMI 监控，默认 1小时显式进行 Full GC**            |
| `-XX:+PrintGC`                                     | `-Xlog:gc`                                         | [GC (Allocation Failure)  9216K->7930K(23552K), 0.0030461 secs] | 打印 GC **基本信息**                                         |
| `-XX:+PrintGCDetails`                              | `-Xlog:gc*`                                        | [GC (Allocation Failure) [PSYoungGen: 9216K->996K(10240K)] 9216K->7938K(23552K), 0.0028309 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] | 打印 GC **详细信息**                                         |
| -XX:+PrintGCApplicationStoppedTime                 | **已废弃**                                         | Total time for which application threads were stopped: 0.0468229 seconds | 打印 GC 的 `STW` 时间                                        |
| -XX:+PrintGCApplicationConcurrentTime              | **已废弃**                                         | Application time: 2.0373444 seconds                          | 打印 GC 回收前程序未中断的执行时间                           |
| -XX:+PrintTLAB                                     | **已废弃**                                         | TLAB: gc thread: 0x00007fce4d0d2000 [id: 16387] desired_size: 184KB slow allocs: 0  refill waste: 2944B alloc: 0.99996     9216KB refills: 1 waste 99.9% gc: 188584B slow: 0B fast: 0B<br/>TLAB totals: thrds: 3  refills: 50 max: 47 slow allocs: 3 max 3 waste:  2.9% gc: 244640B max: 188584B slow: 23784B max: 23720B fast: 1096B max: 1096B | 查看 `TLAB` 空间的使用情况                                   |
| -XX:TLABWasteTargetPercent=1                       | -XX:TLABWasteTargetPercent=1                       | **默认值 1%**                                                | TLAB 占 `Eden Space` 的百分比                                |
| -XX:+DoEscapeAnalysis                              | -XX:+DoEscapeAnalysis                              |                                                              | 开启 `逃逸分析`                                              |
| `-XX:+PrintHeapAtGC`                               | `-Xlog:gc+heap=trace`                              |                                                              | 打印 GC 前后的详细堆栈信息                                   |
| `-Xdebug`                                          | `-Xlog:gc=debug`                                   |                                                              | 根据 GC 级别打印日志                                         |
|                                                    | -Xlog:gc=debug:file=gc.txt                         |                                                              |                                                              |
| `-XX:+PrintTenuringDistribution`                   | `-Xlog:gc+age*=trace`                              | Desired survivor size 87359488 bytes, new threshold 4 (max 15)<br/>- age   1:    9167144 bytes,    9167144 total<br/>- age   2:    9178824 bytes,   18345968 total<br/>- age   3:   16101552 bytes,   34447520 total<br/>- age   4:   21369776 bytes,   55817296 total | `GC Event` 之后打印 `Survivor` 对象情况                      |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 导出日志及对转储文件**                     | **JVM 9 ~ N 导出日志及对转储文件**                 | **JVM 导出日志及对转储文件格式**                             | **JVM 导出日志及对转储文件说明**                             |
| **-XX:ErrorFile=/path/error.log**                  | **-XX:ErrorFile=/path/error.log**                  |                                                              | 在指定路径输出应用程序内的错误日志                           |
| `-XX:+UseGCLogFileRotation`                        | `-XX:+UseGCLogFileRotation`                        |                                                              | 开启日志滚动策略                                             |
| `-XX:NumberOfGCLogFiles=100`                       | `-XX:NumberOfGCLogFiles=100`                       |                                                              | **最多存储 100 个日志文件**                                  |
| `-XX:GCLogFileSize=50m`                            | `-XX:GCLogFileSize=50m`                            |                                                              | **每个日志文件大小为 50 MB**                                 |
| `-Xloggc:/path/gc.log`                             | `-Xlog:gc:/path/gc.log`                            | `.log`                                                       | 在指定路径输出后缀为 `.log` 日志文件                         |
| -XX:+HeapDumpOnOutOfMemoryError                    | -XX:+HeapDumpOnOutOfMemoryError                    |                                                              | 出现 `OutOfMemoryError` 情况下将堆转储到物理文件中           |
| -XX:HeapDumpPath=/path/*.hprof                     | -XX:HeapDumpPath=/path/*.hprof                     | `.hprof`                                                     | 在指定路径输出后缀为 `.hprof` 堆转储文件                     |
| -XX:OnOutOfMemoryError="< cmd args >;< cmd args >" | -XX:OnOutOfMemoryError="< cmd args >;< cmd args >" |                                                              | 出现 `OutOfMemoryError` 情况下，执行用于发出紧急命令以在出现内存不足错误时执行；在cmd args的空格中应使用正确的命令。例如立即重启服务器 `-XX:OnOutOfMemoryError="shutdown -r"` |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 线程相关**                                 | **JVM 9 ~ N 线程**                                 |                                                              | **JVM 线程相关说明**                                         |
| -XX:+UseSpinning                                   | -XX:+UseSpinning                                   | **默认开启**                                                 | 开启自旋锁                                                   |
| -XX:PreBlockSpin=10                                | -XX:PreBlockSpin=10                                | **默认值 10**                                                | 设置自旋锁自旋次数                                           |
| -XX:+UseBiasLocking                                | -XX:+UseBiasLocking                                | **默认开启**                                                 | 开启 JVM `偏向锁`                                            |
| -XX:MaxJavaStackTraceDepth=1024                    | -XX:MaxJavaStackTraceDepth=1024                    |                                                              | `Stack Memory` 栈深                                          |
| -XX:+UseLWPSynchronization                         | -XX:+UseLWPSynchronization                         |                                                              | 使用轻量级进程替换线程同步                                   |
| -XX:+UseBoundTherads                               | -XX:+UseBoundTherads                               |                                                              | 绑定所有的用户线程到内核线程，减少线程进入到饥饿状态次数     |
|                                                    |                                                    |                                                              |                                                              |
| **JVM 8 字符串优化**                               | **JVM 9 ~ N  字符串优化**                          | **JVM 8 字符串优化格式**                                     | **JVM 字符串说明**                                           |
| -XX:+UseStringCache                                | -XX:+UseStringCache                                |                                                              | 启用常用分配的字符串的缓存                                   |
| -XX:+UseCompressedStrings                          | -XX:+UseCompressedStrings                          |                                                              | 对可以使用纯ASCII表示的字符串使用byte []                     |
| -XX:+OptimizeStringConcat                          | -XX:+OptimizeStringConcat                          |                                                              | 尽可能优化字符串连接操作                                     |
| -XX:AutoBoxCacheMax=127                            | -XX:AutoBoxCacheMax=127                            | **默认 [-128, 127]**                                         | 设置自动拆装箱 `IntegerCache` 的**静态内部类**缓存，在此范围内的两个对象是**内存地址是相同的**。对 Long 无效。**旨在减少内存开销**，只在 `Server` 模式有效。 |
|                                                    |                                                    |                                                              |                                                              |

### 4.1 确定内存使用率（活动数据大小）

![Tuning-size](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Tuning-size.png)

`应用程序运行`可以分为三个阶段：

1.  **初始化阶段**
    -   JVM加载应用程序并初始化应用程序的主要模块和数据
2.  **稳定性阶段**
    -   该应用程序已经运行了很长时间，并且已经进行了压力测试。每个性能参数处于稳定状态。使用 JIT 编译已执行并预热了核心功能。
3.  **报告阶段**
    -   在最后的报告阶段，将进行一些基准测试以生成相应的报告。

**JVM 调优必须基于`稳定性阶段`，在确定内存使用率这一步骤时，主要用于`计算活动数据的大小`，因此该应用程序必须满足两个要求：**

1.  稳定性阶段使用**默认的 JVM 参数**，不手动干预。
2.  发生 `Full GC Event` 时确保程序出于稳定阶段。

#### 4.1.1 解析 GC Logs 格式

> **日志中涉及 `Young Gen` 大小指的是 Eden Space 加上一个 Survivor Space 的空间**

`GC` 回收的对象主要是 JVM 的 `Heap Memory` 和 `Metaspace`。所以在执行测试时，使用 `默认的 JVM 参数`，先计算活动数据的大小。

> **程序运行参数：**-Xmx20m -Xms20m -Xmn10m -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:MaxTenuringThreshold=1

```java
public class GC_Example {

    private static final int _1M = 1024 * 1024;

    public static void main(String[] args) throws InterruptedException {
        while (true) {
	          Thread.sleep(500);
            byte[] byte1 = new byte[3 * _1M];
        }
    }
}
```

**上边程序运行后，获取 GC Log 片段**

```shell
18.217: [GC (Allocation Failure) [PSYoungGen: 6472K->224K(9216K)] 16634K->10426K(19456K), 0.0003036 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
18.218: [Full GC (Ergonomics) [PSYoungGen: 224K->0K(9216K)] [ParOldGen: 10202K->2322K(10240K)] 10426K->2322K(19456K), [Metaspace: 8968K->8968K(1056768K)], 0.0055139 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
```

**Minor GC 分段解析**

`18.217: [GC (Allocation Failure) `

- **18.217** 
  - 表示 GC 自 JVM 启动后发生的时间

- **GC (Allocation Failure)** 
  - **GC** 表示发生了 GC Event
  - **(Allocation Failure)**  表示内存分配失败导致的 GC Event 原因

`[PSYoungGen: 6472K->224K(9216K)] 16634K->10426K(19456K), 0.0003036 secs]`

- **[PSYoungGen：a -> b (c) ]**
  - **PSYoungGen** 表示 `YoungGen` 使用的是  `Parallel Scavenge(多线程垃圾收集器)`
  - **a** 表示 GC Event **前**的 `Eden Space + Survivor Space` 
  - **b** 表示 GC Event **后**的 `Eden Space + Survivor Space`
  - **c** 表示 `Young Gen` 整个空间大小

`16634K->10426K(19456K)`

- **x -> y (z)**
  - **x** 表示 GC Event **前**的整个 `Young Gen + Old Gen` 占用空间
  - **y** 表示 GC Event **后**的整个 `Young Gen + Old Gen` 占用空间
  - **z** 表示整个 `Heap Memory` 大小，即 `Young Gen + Old Gen` 的空间
  - 计算 `Old Gen`  大小，则 `z - c = Old Gen`

`0.0003036 secs]`

- 此次 GC 消耗的时间

`[Times: user=0.00 sys=0.00， real=0.00 secs]`

- **user** 表示用户状态消耗的 CPU 时间
- **sys** 表示内核状态小号的 CPU 时间
- **real** 表示操作从开始到结束所经过的 WallClock Time（墙上时钟时间，表示系统中所有进程运行的时钟总量）

**Full GC 分段解析**

`18.218: [Full GC (Ergonomics)`

- **18.218**
  - 表示 GC 自 JVM 启动后发生的时间
- **Full GC (Ergonomics)**
  - 发生了 `Full GC Event` ，发生原因是 `Ergonomics`。该原因是 JVM 内部自动调整的结果。[Oracle 关于 Ergonomics 文档](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html)
  - 发生 Full GC 的原因有很多，常见的有以下几个：
    - **Allocation Failure** 内存分配失败
    - **System.gc()** 手动调用 GC
    - **Ergonomics** JVM自动调整
    - **Metadata GC Threshold** 元空间达到阈值

`PSYoungGen: 224K->0K(9216K)`

- 参考 **Minor GC 分段解析**

`[ParOldGen: 10202K->2322K(10240K)]` 

- **[ParOldGen：e -> f (g) ]**
  - **ParOldGen** 表示使用的 `Parallel Old Collector` ，该 GC Event 发生在 `Old Gen`
  - **e** 表示 Full GC Event **前** `Old Gen` 占用空间
  - **f** 表示 Full GC Event **后** `Old Gen` 占用空间
  - **g** 表示 `Old Gen` 整个空间大小

`10426K->2322K(19456K)`

- **x -> y  (z)**
  - **x** 表示 Full GC Event **前**的整个 `Young Gen + Old Gen` 占用空间
  - **y** 表示 Full GC Event **后**的整个 `Young Gen + Old Gen` 占用空间
  - **z** 表示整个 `Heap Memory` 大小，即 `Young Gen + Old Gen` 的空间
  - 计算 `Old Gen`  大小，则 `z - c = Old Gen`

`[Metaspace: 8968K->8968K(1056768K)]`

- `Metaspace` 属于堆外空间，使用的**本地内存**。
- **t -> m (n)**
- **t** 表示 Full GC Event **前** `Metasapce` 占用空间
- **m** 表示 Full GC Event **后** `Metasapce` 占用空间
- **n** 表示 JVM Metapace 占用的本地内存空间

`[Times: user=0.02 sys=0.00, real=0.00 secs] `

- 参考 **Minor GC 分段解析**

#### 4.1.2 确定内存大小

**JVM Heap 参数优化参考，以 Old Gen 大小为基准进行调整，推荐设置大小 3~4倍，1.2~1.5 倍**

假定经过多次的压测稳定阶段，当发生 `Full GC Event` 时  `OldGen` 的内存使用了 **102400K（100 MB）**。 

| 内存分代        | 参数                                            | 说明                                                         | 大小                                                       |
| --------------- | ----------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| **Heap 总大小** | **-Xms400m<br />-Xmx268m**                      | 初始的 Heap 的大小<br />Heap 的最大值                        | **Old Gen × 4 = 400 MB**                                   |
| **YoungGen**    | **-Xmn150m**                                    | Young Gen 初始值                                             | **Old Gen × 1.5 = 150 MB**                                 |
| **OldGen**      | 无需设置                                        | Old Gen（不需要设置）                                        | **Heap - YoungGen = 250 MB**                               |
| **Metaspace**   | -XX:MetaspaceSize=N<br />-XX:MaxMetaspaceSize=N | 每次触发 `Full GC` 扩容的阈值，**不是初始值**<br />`Metaspace` 最大值，建议设置最大值，因为默认是系统内存值，容易导致系统内存不够。 | 该值正常来讲是 `Metaspace` 在压测时稳定平均值的 **1.5** 倍 |

**根据上述表格，JVM 优化后，还可以切换不同的 GC 以获得更好的性能。**

### 4.2 延迟调整

确定应用程序的活动数据大小后，我们需要执行延迟调整。由于此时 `Heap Memory` 大小和延迟无法满足应用程序要求，因此我们需要根据应用程序的实际要求来调试应用程序。

在此阶段，我们可能需要再次优化 `Heap` 配置，评估 `GC` 的**持续时间和频率**，并确定是否有必要切换到其他垃圾收集器。

#### 4.2.1 系统延迟指标

**延迟调整之前，我们需要知道什么是系统延迟要求，以及可以针对延迟调整哪些指标**

1.  **可接受的应用程序平均 downtime**
    -   此时间将与测得的 `Minor GC` 持续时间进行比较
2.  **可接受的 `Minor GC` 频率**
    -   将 `Minor GC` 频率与容忍值进行比较
3.  **可接受的最大 pause time**
    -   在最坏的情况下，将最大 `pause time` 与 `Full GC 持续时间` 进行比较
4.  **可接受的最大暂停发生频率**
    -   这基本上是 `Full GC` 的频率

在上述指标中，请特别注意 **1** 和 **3**。这两个指标对于用户体验至关重要。

#### 4.2.2 优化 Young Gen

>   0.291: [GC (Allocation Failure) [PSYoungGen: 33280K->5088K(38400K)] 33280K->24360K(125952K), 0.0365286 secs] [Times: user=0.11 sys=0.02, real=0.04 secs] 
>   0.446: [GC (Allocation Failure) [PSYoungGen: 38368K->5120K(71680K)] 57640K->46240K(159232K), 0.0456796 secs] [Times: user=0.15 sys=0.02, real=0.04 secs] 
>   0.829: [GC (Allocation Failure) [PSYoungGen: 71680K->5120K(71680K)] 112800K->81912K(159232K), 0.0861795 secs] [Times: user=0.23 sys=0.03, real=0.09 secs]

从上面的GC日志中，可以简单统计下**分配率**。**分配率是在每个时间单位内分配的内存使用量。分配率过高可能意味着应用程序性能出现问题。在 JVM 上运行时，此问题将造成大量开销的垃圾回收。**

>   **分配值 = x -  y**
>
>   **分配率 = 分配值 / GC 间隔时间**

| GC 事件    | 启动时间 | 暂停时间       | Young 前 | Young 后 | Heap 前 `x` | Heap 后 `y` | 分配值    | 分配率        |
| ---------- | -------- | -------------- | -------- | -------- | ----------- | ----------- | --------- | ------------- |
| 1 Minor GC | 291 ms   | 0.0365286 secs | 33,280KB | 5,088KB  | 33,280KB    | 24,360KB    | 33,280KB  | **114MB/sec** |
| 2 Minor GC | 446 ms   | 0.0456796 secs | 38,368KB | 5,120KB  | 57,640KB    | 46,240KB    | 33,280KB  | **215MB/sec** |
| 3 Minor GC | 829 ms   | 0.0861795 secs | 71,680KB | 5,120KB  | 112,800KB   | 81,912KB    | 66,560KB  | **174MB/sec** |
| **全部**   | 829ms    | 0.168,387,7 s  |          |          |             |             | 133,120KB | **161MB/sec** |
| **平均**   | 276ms    | 0.0561s        |          |          |             |             |           |               |

上述日志 GC 平均时间 **56 ms**，且平均 **276ms** 发生一次。

**如果根据 `指标 1` 可以接受的 50ms，则可以对减小 `Young Gen` 内存空间 10% 大小，尽可能保留 `Old Gen` 和 `Metaspace` 大小，通过该步骤的不断重复进行最优化。**

因为 `Young Gen` 内存空间越大 `Minor GC` 时间越长且频率越低，内存空间越小 `Minor GC` 时间越短且频率越高。

-   **减小暂停时间，需要减小 `Young Gen` 内存空间**
-   **降低暂停频率，需要增大 `Young Gen` 内存空间**

**JVM 参数可以参考确定内存大小的参数进行优化，需要注意分配率的变化。**

#### 4.2.3 优化 Old Gen

>   0.291: [GC (Allocation Failure) [PSYoungGen: 33280K->5088K(38400K)] 33280K->24360K(125952K), 0.0365286 secs] [Times: user=0.11 sys=0.02, real=0.04 secs] 
>   0.446: [GC (Allocation Failure) [PSYoungGen: 38368K->5120K(71680K)] 57640K->46240K(159232K), 0.0456796 secs] [Times: user=0.15 sys=0.02, real=0.04 secs] 
>   0.829: [GC (Allocation Failure) [PSYoungGen: 71680K->5120K(71680K)] 112800K->81912K(159232K), 0.0861795 secs] [Times: user=0.23 sys=0.03, real=0.09 secs]

可以参考  **优化 Young Gen** 方案进行优化。也可以使用根据对象分配率来优化 `Old Gen`。

如果没有 `Full GC` 日志，可以使用 `PSYoungGen` 的日志分析。

1.  `Old Gen 已用空间增长 u = y - b`，此处的**字母代表的值**，具体参考 **Minor GC 分段解析**。

2.  `Minor GC` 平均发生时间为 `times`，则 `Young Gen` 向 `Old Gen` 的对象提升率为 `avg = u(平均值) / times`。

3.  `Old Gen Size` 空间存储满的时间 = `Old Gen Size / avg` 

以上两种方法都可以算出最差的 Full GC 频率。

### 4.3 吞吐量调整

>   **应用程序运行时间总长 / 程序运行时间总长 = 吞吐量**
>
>   GC 运行时间总长越低，吞吐量越高。

上述内存使用率、延迟经过调整后，如果吞吐量达到要求，则调整结束。如果还有一定差距，则可以适当调整参数，增加或减少应用程序的线程用时与程序总用时的占比。

#### 4.4 JVM 调优参数模板

```shell
-d64
-server
-Dsun.rmi.dgc.client.gcInterval=3600000
-Dsun.rmi.dgc.server.gcInterval=3600000
-Xmx2048m 
-Xms2048m 
-Xmn512m
-XX:MetaspaceSize=50m
-XX:MaxMetaspaceSize=1024m
-XX:MaxTenuringThreshold=15
-XX:MaxGCMinorPauseMillis=1
-XX:MaxGCPauseMillis=5
-XX:+UseParallelGC
-XX:+AggressiveOpts
-XX:+PrintGCDateStamps
-XX:+PrintGCDetails
-XX:+PrintTenuringDistribution
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/*.hprof
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=100
-XX:GCLogFileSize=50m
-Xloggc:/path/gc.log
```

