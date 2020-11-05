# JVM 性能调优指南
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
    -   **jstatd** - 实时本地、远程监控 JVM 状态，需要在服务器机器设置一个后缀为 `.policy` 的文件。
-   **Troubleshooting Tools**
    -   **jinfo** - 查看 JVM 配置参数，还可以动态设置部分参数值。`jinfo` 使用时需要 `attach` 到目标 JVM 上。
    -   **jhat** - 用于解析堆、支持生成堆信息文件，并对转储文件进行展示分析，服务启动后，可以通过 http://127.0.0.1:7000/ 查看。
    -   **jmap** - 打印 JVM 堆概要信息，包括堆配置、新生代、老生代信息，支持**导出文件**，并结合 `jvisualvm`、`MAT`、`JProfile` 等工具进行分析。当使用 `jamp -histo:live pid` 携带 `:live` 命令只打印活动类信息，会触发 GC，导致 `STW`，生产环境慎用。
    -   jsadebugd
    -   **jstack** - 查看 JVM 进程内当前时刻的线程快照，定位线程停顿、死锁等长时间等待的问题，支持导出线程相关信息。
-   **Troubleshooting, Profiling, Monitoring and Management Tools**
    -   **jcmd**- 用于将诊断命令请求发送到 JVM，不支持远程连接 JVM。 
    -   **jconsole** - 实时本地、远程监控 CPU、JVM 内存、类加载、垃圾收集、线程（检测死锁）和类等的监控，是一个基于 JMX（java management extensions）的 GUI 性能监测工具。
    -   **jvisualvm** - `jmap、jinfo、jstat、jstack、jconsole` 功能合集，支持多种功能插件，因为 `jvisualvm` 不仅支持 JMX 还支持 Jvmstat、Attach API 和 SA 等监测技术，还可以记录有关 JVM 实例的数据，并将该数据保存到本地系统，便于分析，支持**内存泄漏检测**。
    -   **jmc** - 相对 `jconsole`、`jvisualvm` 可以展示更多的信息，也是 GUI 监测工具，UI 更美观，在 `JDK 8u261` 被移除，[Oracle JDK 8u261更新文档](https://www.oracle.com/java/technologies/javase/8u261-relnotes.html)，可在 [Oracle 官方](https://www.oracle.com/java/technologies/javase/jmc-install.html) 单独下载。

以上基本介绍来自 [Oracle Tools 文档](https://docs.oracle.com/javase/8/docs/technotes/tools/)

### 2.1 jsp tools

[Oracle jps command](https://docs.oracle.com/en/java/javase/13/docs/specs/man/jps.html)

| JPS 命令            | 显示结果                                   |
| ------------------- | ------------------------------------------ |
| Jps                 | 显示 vmid、启动类名称                      |
| jps -m              | 显示 vmid、传递 main 方法的参数            |
| Jps -l              | 显示 vmid、启动类完成包名或 jar 完整路径名 |
| jps -v              | 显示 vmid、传递给 jvm 参数                 |
| jps -V              | 显示 vmid、启动类名、jar 文件名            |
| m、l、v、V 任意组合 | 显示结果组合                               |

### 2.2 jstat tools

[Oracle jstat command](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jps.html)

### JVisualvm Tools

[Oracle 远程链接文档](https://docs.oracle.com/javase/1.5.0/docs/guide/management/agent.html#PasswordAccessFiles)

#### 远程无认证连接

1.  服务器启动 jar

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

#### 远程安全认证连接

1.  **服务器**进入路径 `JRE_HOME/lib/management/`
2.  拷贝 `jmxremote.password.template` 为 `jmxremote.password` 并打开 `monitorRole` 和 `controlRole` 注释，`monitorRole` 只可以监控，`controlRole` 可以在本地服务器进行一些操作，比如堆转储，GC 等操作。

```shell
# The "monitorRole" role has password "QED".
# The "controlRole" role has password "R&D".
monitorRole QED
controlRole R&D
```

3.  使用 JVM 参数启动 `***.jar`

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



