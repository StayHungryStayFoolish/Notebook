# JVM 性能调优指南
## JVM 性能监测工具

### Application Performance Management(APM)

#### APM 与 Metrics 两个体系的区别

两个体系都属于监控系统，但却有很大的区别，此处只做简单记录。

**APM** 体系都是基于 [Google Dapper](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf) 的一篇论文实现的，旨在**监测程序内部执行过程和服务间链路调用情况，更利于深入剖析关于代码层面的问题，侧重于 Tracing 方向。**

-   APM 体系开源项目
    1.  [Pinpoint](https://github.com/pinpoint-apm/pinpoint)
    2.  [SkyWalking](https://github.com/apache/skywalking)
    3.  [Glowroot](https://github.com/glowroot/glowroot)
    4.  [Zipkin](https://github.com/openzipkin/zipkin)
    5.  [CAT](https://github.com/dianping/cat)

推荐使用 `Pinpoint` 或者 `SkyWalking`。因为这两个都是通过字节码注入，对代码完全无侵入。并且 `SkyWalking` 是国内大神**吴晟**开源的，目前已在 `Apache` 孵化，且社区活跃。`Pinpoint` 是韩国公司开源，社区活跃度与沟通上相对差一些。具体性能对比可自行了解。`Glowroot` 对代码也是无侵入的。 `Zipkin` 经常与 `Spring Cloud Slueth` 组合，完成全链路监控的作用，并且 `Zipkin` 还可以和 `ELK` 组合来记录日志和展示。`CAT` 是大众点评开源。

**Metrics** 体系旨在**监控服务器硬件指标与系统服务运行状态等**。其中 `Prometheus` 基于 `Google Borgmon` 内部原理设计而来。

-   Metrics 体系开源项目
    1.  [Prometheus](https://github.com/prometheus/prometheus)
    2.  [Zabbix](https://github.com/zabbix/zabbix)
    3.  [Kamon](https://github.com/kamon-io/Kamon)

`Prometheus` 通常和 `Grafana` 配合使用，因为 `Grafana` 只负责绘图，并且 UI 比较酷炫。

## JVM 常用命令和参数
