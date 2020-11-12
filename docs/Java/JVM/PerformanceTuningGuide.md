# JVM 性能调优指南
**Java 应用性能优化依次可以划分为 4 个层级**

1.  **应用层**
    -   优化代码结构，避免线程阻塞、死锁等情况
2.  **数据库层**
    -   优化 SQL 语句，避免数据库全表扫描、死锁等情况
3.  **架构层**
    -   在理解架构机制的情况下进行最优配置，提高计算、存储能力，降低系统负载
4.  **JVM 层**
    -   通过对应用程序的压测，不断调整  JVM 参数，最终达到低延迟、高吞吐量、最优内存使用率三个性能指标

Java 应用程序影响性能的因素非常多，比如磁盘、内存、IO等因素。以上 4 个层级优化难度是依次递增的，架构层优化对系统性能影响最大。

**本文假定前三者已经优化之后，仍需提高性能，那么 JVM 则是最后一次提升性能的关键操作，调整的最终目标是以最低的硬件消耗成本使应用程序具有更高的性能。。**

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

## 3.  JVM 性能调优

上边介绍了各种 JVM 监测工具的相关配置和部分参数。接下来将介绍如何对 JVM 进行性能调优。

调整的最终目标是以最低的硬件消耗成本使应用程序具有更大的吞吐量。JVM 调整也不例外。

JVM 调优主要涉及优化**GC(垃圾收集器)**以获得更好的收集性能，以便在 JVM 上运行的应用程序可以具有更大的吞吐量，同时使用更少的内存并具有更低的延迟。

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

### 3.3 调优过程

**JVM 调整涉及连续的配置优化和基于性能测试结果的多次迭代。**在满足每个所需的系统指标之前，每个先前的步骤可能会经历多次迭代。在某些情况下，为了满足特定指标，可能需要多次调整先前的参数，从而需要再次测试所有先前的步骤。

此外，调整通常从满足应用程序的`内存使用`要求开始，然后是`延迟时间`和`吞吐量`。调整应遵循以下步骤顺序。

#### 3.3.1 确定内存使用率

`应用程序运行`可以分为三个阶段：

1.  **初始化阶段**
    -   JVM加载应用程序并初始化应用程序的主要模块和数据
2.  **稳定性阶段**
    -   该应用程序已经运行了很长时间，并且已经进行了压力测试。每个性能参数处于稳定状态。使用 JIT 编译已执行并预热了核心功能。
3.  **报告阶段**
    -   在最后的报告阶段，将进行一些基准测试以生成相应的报告。

`GC` 回收的对象主要是 JVM 的 `Heap Memory` 和 `Metaspace`。所以在执行测试时，使用 `默认的 JVM 参数`，先计算活动数据的大小。

```shell
# 输出GC的时间戳（以JVM启动到当期的总时长的时间戳形式）
-XX:+PrintGCTimeStamps
# 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintGCDateStamps 
# 以上 2 个打印时间的参数在 JDK 1.8.0_191 已废弃
# 使用 ParallelGC，在 JDK 1.8.0_191 版本，虽然使用 java -XX:+PrintCommandLineFlags -version 命令查看显示默认使用 ParallelGC，但是如果不使用该参数会使用 G1GC，估计 JDK 在版本升级后的遗留问题
 -XX:+UseParallelGC
# 打印 GC ，在当前版本已经提示 -XX:+PrintGCDetails 将要废弃
-Xlog:gc*
# 并在当前位置输出 gc.log
-Xloggc:gc.log 
# 在进行GC的前后打印出堆的信息，如果 -XX:+PrintHeapAtGC 不能使用替换为一下参数
-Xlog:gc+heap=trace
# 当出现 OOM 进行堆转储
-XX:+HeapDumpOnOutOfMemoryError 
# 在当前路径下生成后缀为 .hprof 堆转储文件，此处可以指定一个路径
-XX:HeapDumpPath=
```

**应用程序出于稳定阶段的 GC Logs 分析（Minor GC & Full GC）**

使用 `-Xmx100m -Xlog:gc* -XX:+UseParallelGC -Xlog:gc+heap=trace` 参数打开 GC 日志，此处为了快速获取 GC 日志，所以限制了 Heap 最大内存为 **100MB**。

```shell
# 整个日志开头的 s 代表自 JVM 启动的时间
# 初始 Heap 大小
[0.004s][trace][gc,heap]   Initial heap size 104857600
# 最小 Heap 大小
[0.004s][trace][gc,heap]   Minimum heap size 6815736
[0.010s][debug][gc,heap] Minimum heap 8388608  Initial heap 104857600  Maximum heap 104857600
[0.010s][trace][gc,heap] 1: Minimum young 1572864  Initial young 34603008  Maximum young 34603008
[0.010s][trace][gc,heap] Minimum old 524288  Initial old 70254592  Maximum old 70254592
# 使用 Parallel GC
[0.010s][info ][gc     ] Using Parallel

# Heap 大小 = Young Gen（33 MB） + Old Gen（67 MB） = 100 MB
# Young Gen（33 MB） = Eden Space(25 MB) + from[Survivor0]（4 MB） + to[Survivor1]（4 MB）

[0.010s][info ][gc,heap,coops] Heap address: 0x00000007f9c00000, size: 100 MB, Compressed Oops mode: Zero based, Oop shift amount: 3
# PSYoungGen total = 29696K = 29 MB(次数 YoungGen 只加了一个 Survivor) 
[122.009s][debug][gc,heap      ] GC(0) Heap before GC invocations=1 (full 0): PSYoungGen      total 29696K, used 25600K [0x00000007fdf00000, 0x0000000800000000, 0x0000000800000000)
# eden space 25600K = 25 MB，目前已经使用了 100%，接下来再分配对象到 Eden，肯定会出现 Allocation Failure（分配内存失败），发生 Minor GC Event
[122.010s][debug][gc,heap      ] GC(0)   eden space 25600K, 100% used [0x00000007fdf00000,0x00000007ff800000,0x00000007ff800000)
# from space 4096K = 4 MB，0% 使用
[122.010s][debug][gc,heap      ] GC(0)   from space 4096K, 0% used [0x00000007ffc00000,0x00000007ffc00000,0x0000000800000000)
# to space 4096K = 4 MB，0% 使用
[122.010s][debug][gc,heap      ] GC(0)   to   space 4096K, 0% used [0x00000007ff800000,0x00000007ff800000,0x00000007ffc00000)
# ParOldGen 68608K = 67 MB，目前使用 0K
[122.010s][debug][gc,heap      ] GC(0)  ParOldGen       total 68608K, used 0K [0x00000007f9c00000, 0x00000007fdf00000, 0x00000007fdf00000)
[122.010s][debug][gc,heap      ] GC(0)   object space 68608K, 0% used [0x00000007f9c00000,0x00000007f9c00000,0x00000007fdf00000)
# Metaspace 
# used 代表加载的类的大小
# capacity 代表当前分配块的元数据的空间，被实际分配的 Chunk 总和 
# committed 代表空间块的数量，被 commit 的 Chunk 总和
# reserved 代表元数据的保留大小，JVM 启动时根据参数和操作系统预留的内存大小
[122.010s][debug][gc,heap      ] GC(0)  Metaspace       used 14034K, capacity 14512K, committed 14720K, reserved 1062912K
# Metaspace 的 class space 
[122.010s][debug][gc,heap      ] GC(0)   class space    used 1438K, capacity 1629K, committed 1664K, reserved 1048576K
# 因为 YoungGen 的内存分配失败，发生 MinorGC Event
[122.010s][info ][gc,start     ] GC(0) Pause Young (Allocation Failure)
# MinorGC 前 YoungGen 使用了 25600K，MinorGC 后使用了 4066K，总大小为 29696K = 29 MB
[122.018s][info ][gc,heap      ] GC(0) PSYoungGen: 25600K->4066K(29696K)
# MinorGC 前 OldGen 使用了 0K，MinorGC 后，OldGen 增加了 992K，总大小为 68608K = 67 MB
[122.018s][info ][gc,heap      ] GC(0) ParOldGen: 0K->992K(68608K)
# Meataspace 前后没有变化，总大小约 1G，因为没有设置 -XX:MaxMetaspaceSize
[122.018s][info ][gc,metaspace ] GC(0) Metaspace: 14034K->14034K(1062912K)
# YoungGen 发生了 MinorGC，暂停了 8.594ms(STW 的时间)
[122.018s][info ][gc           ] GC(0) Pause Young (Allocation Failure) 25M->4M(96M) 8.594ms
# User 代表用户状态消耗的 CPU 时间
# Sys 代表内核状态小号的 CPU 时间
# Real 代表操作从开始到结束所经过的 WallClock Time（墙上时钟时间，表示系统中所有进程运行的时钟总量）
[122.018s][info ][gc,cpu       ] GC(0) User=0.01s Sys=0.01s Real=0.00s
# GC 后的日志，使用 -Xlog:gc+heap=trace 打开
[122.018s][debug][gc,heap      ] GC(0) Heap after GC invocations=1 (full 0): PSYoungGen      total 29696K, used 4066K [0x00000007fdf00000, 0x0000000800000000, 0x0000000800000000)
[122.018s][debug][gc,heap      ] GC(0)   eden space 25600K, 0% used [0x00000007fdf00000,0x00000007fdf00000,0x00000007ff800000)
[122.018s][debug][gc,heap      ] GC(0)   from space 4096K, 99% used [0x00000007ff800000,0x00000007ffbf8a58,0x00000007ffc00000)
[122.018s][debug][gc,heap      ] GC(0)   to   space 4096K, 0% used [0x00000007ffc00000,0x00000007ffc00000,0x0000000800000000)
[122.018s][debug][gc,heap      ] GC(0)  ParOldGen       total 68608K, used 992K [0x00000007f9c00000, 0x00000007fdf00000, 0x00000007fdf00000)
[122.018s][debug][gc,heap      ] GC(0)   object space 68608K, 1% used [0x00000007f9c00000,0x00000007f9cf8030,0x00000007fdf00000)
[122.018s][debug][gc,heap      ] GC(0)  Metaspace       used 14034K, capacity 14512K, committed 14720K, reserved 1062912K
[122.018s][debug][gc,heap      ] GC(0)   class space    used 1438K, capacity 1629K, committed 1664K, reserved 1048576K
# 下次 GC 开始的 Heap 数据
[342.724s][debug][gc,heap      ] GC(1) Heap before GC invocations=2 (full 0): PSYoungGen      total 29696K, used 29666K [0x00000007fdf00000, 0x0000000800000000, 0x0000000800000000)

# 在程序运行一段时间后，发生了 FullGC Event
19.779s][info ][gc,start       ] GC(43) Pause Full (Ergonomics)
# GC Roots 开始标记、整理、压缩三个步骤（不清楚 GC Roots 的请查看 JVM 内存模型 2.垃圾收集机制 2.2.1GC Roots 工作原理章节）
[19.779s][info ][gc,phases,start] GC(43) Marking Phase
[20.049s][info ][gc,phases      ] GC(43) Marking Phase 269.603ms
[20.049s][info ][gc,phases,start] GC(43) Summary Phase
[20.049s][info ][gc,phases      ] GC(43) Summary Phase 0.007ms
[20.049s][info ][gc,phases,start] GC(43) Adjust Roots
[20.049s][info ][gc,phases      ] GC(43) Adjust Roots 0.565ms
[20.049s][info ][gc,phases,start] GC(43) Compaction Phase
[20.102s][info ][gc,phases      ] GC(43) Compaction Phase 52.669ms
[20.102s][info ][gc,phases,start] GC(43) Post Compact
[20.102s][info ][gc,phases      ] GC(43) Post Compact 0.140ms
[20.102s][info ][gc,heap        ] GC(43) PSYoungGen: 25600K->25600K(29696K)
[20.102s][info ][gc,heap        ] GC(43) ParOldGen: 68406K->68406K(68608K)
[20.102s][info ][gc,metaspace   ] GC(43) Metaspace: 7486K->7486K(1056768K)
[20.102s][info ][gc             ] GC(43) Pause Full (Ergonomics) 91M->91M(96M) 323.139ms
[20.102s][info ][gc,cpu         ] GC(43) User=0.50s Sys=0.01s Real=0.33s
[20.102s][debug][gc,heap        ] GC(43) Heap after GC invocations=44 (full 41): PSYoungGen      total 29696K, used 25600K [0x00000007fdf00000, 0x0000000800000000, 0x0000000800000000)
```

假定以上日志是经过多次的压测稳定阶段产生的 `GC` 日志，可以发现 `OldGen` 的内存当使用到了 **68608K（67 MB）** 时产生了 `Full GC`。

**JVM Heap 参数优化参考，以 Old Gen 大小为基准进行调整**

| 内存分代        | 参数                                            | 说明                                                         | 大小                                                       |
| --------------- | ----------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| **Heap 总大小** | **-Xms268m<br />-Xmx268m**                      | 初始的 Heap 的大小<br />Heap 的最大值                        | **Old Gen * 4 = 268 MB**                                   |
| **YoungGen**    | **-Xmn100.5m**                                  | Young Gen 初始值                                             | **Old Gen * 1.5 = 100.5 MB**                               |
| **OldGen**      |                                                 | Old Gen 初始值（不需要设置）                                 | **Heap(256 MB) - YoungGen(100.5 MB) = 167.5 MB**           |
| **Metaspace**   | -XX:MetaspaceSize=N<br />-XX:MaxMetaspaceSize=N | 每次触发 `Full GC` 扩容的阈值，默认值根据系统不同在 14 ~ 20 MB 之间，**不是初始值**<br />`Metaspace` 最大值，建议设置最大值，因为默认是系统内存值，容易导致系统内存不够。 | 该值正常来讲是 `Metaspace` 在压测时稳定平均值的 **1.5** 倍 |

​    





调优需要达到的最高性能要求

1.  最低延迟
    -   一致的 Java 应用响应时间
    -   [LatencyUtils](https://github.com/LatencyUtils/LatencyUtils)
2.  不出现抖动和有问题的垃圾收集暂停现象
    -   消除由停顿和抖动引起的问题。