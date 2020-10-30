# 垃圾收集器

JVM 实际上提供了七个不同的垃圾收集器。每个垃圾收集器在 `Application throughput` 和 `Application pause` 两个指标上衡量会有所不同。`Application throughput`  表示 Java 应用程序运行的速度，`Application pause` 表示垃圾收集器清理未使用的内存空间所需的时间。

#### 2.3.3.1 Serial Garbage Collector(串行垃圾收集器)

-   最简单的 GC 实现方式，基本只使用一个线程工作。如果采用该收集器工作，串行垃圾收集器则拥有程序的所有运行线程。

-   **工作原理**

    -   创建一个单线程来执行垃圾收集。`冻结`所有其他正在运行的应用程序线程，直到垃圾收集操作结束。
    
-   **缺点**

    -   造成应用程序吞吐量降低，增加应用程序暂停时间。

#### 2.3.3.2 **Parallel Garbage Collector(并行垃圾收集器)**

-   `Java 8 默认的垃圾收集器`。

-   设置垃圾收集器在收集时的线程数量

    -   `-XX:+UseParallelGC -XX:ParallelGCThreads=NumberOfThreads -jar GFGApplicationJar.java`
    
-   设置垃圾收集器在收集时的最大暂停时间，单位为毫秒

    -   `-XX:+UseParallelGC -XX:MaxGCPauseMillis=SecInMillisecond -jar GFGApplicationJar.java`
    
-   **工作原理**

    -   并行垃圾收集器与串行工作原理一样。不同的是并行垃圾收集器使用多线程来清理未使用的堆区。
    
-   **存在问题**

    -   在小操作时，依然会暂停应用程序。

#### 2.3.3.3 Concurrent Mark Sweep(CMS) Collector(并发标记扫除收集器)

-   **工作原理**

    -   使用多个线程持续扫描堆内存，对未使用的对象进行标记，然后对标记的对象进行扫除。
    
-   **冻结2种情况**

    1.  在执行垃圾收集的同时，并行的堆区存发生了变化。
    
    2.  标记了一个在 `Old Generation Space(老年代) `的引用对象。
    
-   **存在问题**

    1.  并发模式故障
    
    2.  过多的 GC 时间和 `OutOfMemoryError`
    
    3.  内存碎片（浮动垃圾）
    
    4.  GC 周期内多次 `STW` 暂停
    
-   **Oracle 官方声明已弃用。**
-   [Oracle Docs](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)
  
    -   [Oracle JavaSE HotSpot](https://docs.oracle.com/en/java/javase/11/gctuning/concurrent-mark-sweep-cms-collector.html#GUID-FF8150AC-73D9-4780-91DD-148E63FA1BFF)
    
    -   [OpenJDK  JEP 291](https://openjdk.java.net/jeps/291)
    
    -   [CMS 弃用文章分析](https://www.linkedin.com/pulse/jvm-why-cms-garbage-collector-deprecating-kunal-saxena)

#### 2.3.3.4 Garbage-First Garbage Collector(G1 垃圾收集器)

-   首先在 JDK 7 中引入了 G1 垃圾收集器。起初，它的设计是为了更好地支持较大的堆内存应用，G1 Garbage Collector 是 `Java 9的默认垃圾收集器`。G1 收集器取代了 CMS 收集器，因为它的性能更高效。

-   Java 9 版本以下启动 G1 命令

    -   `XX：+ UseG1GC -jar GFGApplicationJar.java`
    
-   **工作原理**
-   G1 收集器将堆空间分割成多个大小相等的区域。基本上，它主要是为堆大于 `4GB` 的应用程序设计的。它将堆空间划分为多个区域，从 1MB 到 32MB 不等。
  
    -   G1 收集器将堆划分为若干个区域（Region），它仍然属于分代收集器。不过，这些区域的一部分包含新生代，新生代的垃圾收集依然采用暂停所有应用线程的方式，将存活对象拷贝到老年代或者 Survivor 空间。老年代也分成很多区域，G1 收集器通过将对象从一个区域`复制`到另外一个区域，完成了`清理`工作。这就意味着，在正常的处理过程中，G1 完成了`堆的压缩（至少是部分堆的压缩）`，这样也就不会有 CMS 内存碎片问题的存在了。
