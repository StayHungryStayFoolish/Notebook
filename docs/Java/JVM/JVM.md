# JVM 内存模型

## Runtime Data Area

![](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Java-Runtime-data.png)

此文只讨论 JVM 内存模型，如需了解上图全部结构，请查看 [JDK 三大组件](http://notebook.bonismo.ink/#/Java/JVM/JDK)。

 `Runtime Data Area` 是 **OS(操作系统)** 分配给 JVM 用来管理内存分配的。主要包括 `堆压缩` 和 `垃圾对象删除`，并最终提供运行时环境。

