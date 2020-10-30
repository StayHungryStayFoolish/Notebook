# 垃圾收集器

Java 内存管理及其内置的垃圾回收，是该语言最出色的成就之一。它可以使开发人员可以创建对象，而无需担心内存的分配与释放，因为垃圾收集器会自动回收内存以供重用。

Java垃圾收集是一个自动过程，在此过程中，GC 将检查堆上的对象，检查是否仍在引用它们，并释放那些不再需要的对象使用的内存。

**JDK 版本截止在 12 ，JVM 实际上提供了七个不同的垃圾收集器**，相信后续的版本会出现更多优秀的垃圾收集器。每个垃圾收集器在 `Application throughput` 和 `Application pause` 两个指标上衡量会有所不同。`Application throughput`  表示 Java 应用程序运行的速度，`Application pause` 表示垃圾收集器清理未使用的内存空间所需的时间。

![GC](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Java Garbage Collection.png)

## 1. Serial Garbage Collector(串行垃圾收集器)

最简单的 GC 实现方式，基本只使用**一个线程**清理未使用的 `Heap Memory`。如果采用该收集器工作，串行垃圾收集器则拥有程序的所有运行线程。

**工作原理**

创建一个单线程来执行垃圾收集。`冻结`所有其他正在运行的应用程序线程，直到垃圾收集操作结束。

**缺点**

造成应用程序吞吐量降低，增加 `STW` 暂停时间。

## 2. Parallel Garbage Collector(并行垃圾收集器)

`Java 8 默认的垃圾收集器`，也称为**吞吐量垃圾收集器**，与串行垃圾收集器不同的是，并行垃圾收集器使用**多个线程**清理未使用的 `Heap Memory`。

设置垃圾收集器在收集时的线程数量

-   `-XX:+UseParallelGC -XX:ParallelGCThreads=NumberOfThreads -jar GFGApplicationJar.java`

设置垃圾收集器在收集时的最大暂停时间，单位为毫秒

-   `-XX:+UseParallelGC -XX:MaxGCPauseMillis=SecInMillisecond -jar GFGApplicationJar.java`

**工作原理**

并行垃圾收集器与串行工作原理一样。不同的是并行垃圾收集器使用多线程来清理未使用的堆区。

**存在问题**

在小操作时，依然会暂停应用程序。

## 3. Concurrent Mark Sweep(CMS) Collector(并发标记扫除收集器)

**CMS 垃圾收集器被称为并发标记清除垃圾收集器**。该垃圾收集器使用**多个线程来一致地扫描 Heap**以查找未使用的标记对象，然后清除标记的对象。

**工作原理**

使用多个线程持续扫描堆内存，对未使用的对象进行标记，然后对标记的对象进行扫除。

**CMS 垃圾收集器只有在 2 种情况下会冻结应用程序正在运行的线程**

1.  在执行垃圾收集的同时，并行的更改 `Heap Memory`。

2.  在 `Old Generation Space(老年代) ` 标记引用对象。

**存在问题**

1.  并发模式故障

2.  过多的 GC 时间和 `OutOfMemoryError`

3.  内存碎片（浮动垃圾）

4.  GC 周期内多次 `STW` 暂停

**Oracle 官方声明已弃用。**

-   [Oracle Docs](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)
-   [Oracle JavaSE HotSpot](https://docs.oracle.com/en/java/javase/11/gctuning/concurrent-mark-sweep-cms-collector.html#GUID-FF8150AC-73D9-4780-91DD-148E63FA1BFF)
-   [OpenJDK  JEP 291](https://openjdk.java.net/jeps/291)
-   [CMS 弃用文章分析](https://www.linkedin.com/pulse/jvm-why-cms-garbage-collector-deprecating-kunal-saxena)

## 4. Garbage-First Garbage Collector(G1 垃圾收集器)

首先在 JDK 7 中引入了 G1 垃圾收集器。起初，它的设计是为了更好地支持较大的堆内存应用，G1 Garbage Collector 是 `Java 9的默认垃圾收集器`。G1 收集器取代了 CMS 收集器，因为它的性能更高效。

Java 9 版本以下启动 G1 命令

-   `XX：+ UseG1GC -jar GFGApplicationJar.java`

**工作原理**

G1 收集器将堆空间分割成多个大小相等的区域。基本上，它主要是为堆大于 `4GB` 的应用程序设计的。它将堆空间划分为多个区域，从 **1MB** 到 **32MB** 不等。

-   G1 收集器将堆划分为若干个 `堆区域（Heap Region）`，它仍然属于分代收集器。不过，这些区域的一部分包含新生代，新生代的垃圾收集依然采用暂停所有应用线程的方式，将存活对象拷贝到 `Old Gen` 或者  `Survivor` 空间。 `Old Gen` 也分成很多区域，G1 收集器通过将对象从一个区域`复制`到另外一个区域，完成了`清理`工作。这就意味着，在正常的处理过程中，G1 完成了`堆的压缩（至少是部分堆的压缩）`，这样也就不会有 CMS 内存碎片问题的存在了。

## 5. Epsilon Garbage Collection(EGC 垃圾收集器)

JDK 11 引入。`Epsilon` 是一个非操作性的或被动的垃圾收集器。它为应用程序**分配内存**，但它**不收集未使用的对象**。当应用程序用完 Java 堆时，JVM 就会关闭。这意味着 `Epsilon` 垃圾收集器允许应用程序消耗完内存并崩溃。

## 6. Z Garbage Collection(ZGC 垃圾收集器)

**JDK 11 引入，在 JDK 12 之前仍处于实验阶段，仅能在 Linux 64 上运行**。ZGC 可同时执行所有昂贵的工作，而不会将应用程序线程的执行停止 `STW` 超过 **10ms**，这使其适用于要求**低延迟和/或使用非常大堆**的应用程序。根据 `Oracle` 文档，它可以处理**多TB的堆**。ZGC 在其线程中执行其循环。它将平均暂停应用程序 **1ms**。**G1和并行收集器平均大约 200ms**。

**ZGC 通过一种叫做指针着色的技术来利用64位指针。彩色指针可以存储堆上对象的额外信息。这也是它被限制在 64 位 JVM 上的原因之一。**[参考文章](https://www.opsian.com/blog/javas-new-zgc-is-very-exciting/)

**工作原理**

1.  短暂的 `STW` 暂停
    -   检查 `GC Roots`，即指向堆其余部分的局部变量。这些根的总数通常很小，并且不会随负载的大小而缩放，因此 ZGC 的 `STW` 非常短，并且不会随着堆的增长而增加。
2.  并发阶段
    -   通过对象图并检查彩色指针，标记可访问的对象。负载屏障可防止 GC 阶段与任何应用程序活动之间的争用。
3.  重置阶段
    -   移动存活对象，以释放堆中的大部分内存空间，使分配速度更快。当重新定位阶段开始时，ZGC 将堆分成若干页，每次只对一页进行工作。一旦 ZGC 完成所有 `GC Roots` 的移动，其余的重定位就会在并发阶段发生。

**潜在问题**

ZGC 会尝试自行设置线程数，通常是正确的。但是如果 ZGC 的线程太多，它将使您的应用程序的线程一直处于饥饿状态。如果还不够，那么创建垃圾的速度将比 GC 收集垃圾的速度快。

## 7. Shenandoah Garbage Collection(SGC 垃圾收集器)

JDK 12 引入。SGZ 是一个超低暂停时间的垃圾收集器，它通过与正在运行的Java程序并发执行更多的垃圾收集工作来减少GC的暂停时间。CMS 和 G1 都能对实时对象进行**并发标记**。SGC 增加了**并发压实**功能。

SGC 使用内存区域来管理哪些对象不再使用，哪些是存活的，可以进行压实。SGC 还为每个堆对象添加了一个**转发指针**，并使用它来控制对对象的访问。SGC 的设计以并发 CPU 周期和空间换取 `STW` 暂停时间的改善。转发指针使得移动对象变得很容易，但激进的移动意味着 SGC 比其他 GC 使用更多的内存，需要更多的并行工作。但它在非常短暂的停止世界暂停的情况下完成了额外的工作。
