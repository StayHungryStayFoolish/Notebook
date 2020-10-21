# Java 概述

## JDK 三大组件

![JDK](https://gitee.com/bonismo/notebook-img/raw/2a859fbdc77396e1c8e282e29c4ff681a9995072/img/jvm/JDK.png)

### JDK(Java Development Kit)

- `JDK`是开发 Java 引用程序的软件开发环境。`JDK` 包括 `JRE(Java Runtime Environment)`运行时环境、`Java`启动器(解释器/加载器)、`javac`(编译器)、`jar`(归档器)、`javadoc`(文档生成器)以及开发所需其他工具。
  
- JDK 面向的是开发人员，如果只需运行程序，可以单独下载 JRE。
  
### JRE(Java Runtime Environment)

  - `JRE`是构建了一个可以在其中执行 Java 程序的运行时环境。`JRE`提供了执行 Java 应用程序的最低要求。`JRE` 包括 Java 语言常用的 utils、集合框架等核心类库、`JVM(Java Virtual Machine)虚拟机`等。
  
  - **基本工作原理**
  - `JRE` 是 `JDK` 的一部分，在磁盘上。`JRE` 使用 Java 字节码(javac 执行后产生)，将其所需要的 Library 组合，然后启动 `JVM` 来执行。

### JVM(Java Virtual Machine)

[Oracle HotSpot JVM 优化指南](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)

>   **JVM 分为五个基本组件，分别为：**
>
>   1.  ClassLoader SubSystem(类加载器子系统)
>
>   2.  Runtime Data Areas(运行时数据区)
>
>   3.  Execution Engine(执行引擎)
>
>   4.  JNI(Native Method Interface/本地接口)
>
>   5.  Native Method Library(本地方法库)

![JVM-Architecture](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/JVM-Architecture.png)

  - `JVM` 是 `JRE` 的实例，存在于内存中并与内存打交道，是 Java 语言实现跨平台的核心，即 `Write once, run anywhere!`
   
  - **工作原理**
    
    - `JVM` 联合 `JRE`  的 utils、集合框架等，执行 `Java编译器` 编译后的 `*.class` 字节码文件，解释字节码到 `CPU指令集` 或 `OS系统` 调用。
    



## JVM 组件概述

### 1. ClassLoader SubSystem(类加载器子系统)

> **Java 的动态类加载由该系统负责处理。类加载子系统会加载、链接，并在运行时首次引用类的初始化类文件。该系统一共分为三个步骤。**

#### 1.1 Loading(加载)

**类的字节码文件将由此组件加载。BootStrap ClassLoader，Extension ClassLoader 和 Application ClassLoader 是有助于实现该目标的三个 ClassLoader。**

-   **BootStrap ClassLoader**

    -   负责从引导类路径中加载类，仅用于 `rt.jar`。最高优先级将给予此加载程序。

-   **Extension ClassLoader**

    -   负责加载 `ext文件夹(jre\lib)`内的类。

-   **Application ClassLoader**

    -   负责加载应用程序级别的类路径、环境变量路径等。

#### 1.2 Linking(链接)

##### 1.2.1. Verify(验证)

-   字节码验证程序验证生成的字节码是否正确，如果验证失败，将会收到验证错误。

##### 1.2.2 Prepare(准备)

-  为类的`static(静态变量)`分配内存，并设置默认值。

##### 1.2.3  Resolve(解析)

-  将所有符号存储引用替换为 `Method Area` 的原始引用。

#### 1.3 Initialization(初始化)

-   这是 ClassLoading 的最后阶段。在此，所有 `static(静态变量)`将被分配默认值，并且将执行 `静态块`。

`以上 1.1 - 1.3 共 5 个步骤，属于类加载过程。`

### 2. Runtime Data Areas(运行时数据区)

>   **运行时数据区分为五个部分。**

#### 2.1 Method Area(方法区域)

-   所有类级别的数据**（包括静态变量）**都将存储在此区域。每个 JVM 实例都只有一个方法区域，JVM 在启动时创建该区域，为线程间共享。该区域的内存不必是连续的，且可以指定方法区域的大小。

-   如果无法提供方法区域的内存来满足分配需求，JVM 会抛出 `OutOfMemoryError`。

#### 2.2 Heap Area(堆区)
-   所有对象及其对应的实例变量和数组都将存储在此区域。每个 JVM 实例还有一个 `Heap Area(堆区)`。由于 `Method Area` 和 `Heap Area` 为多个线程共享内存，所以`存储的数据不是线程安全的`。

-   如果计算需要的堆多余自动存储管理系统可以提供的堆，JVM 会抛出 `OutOfMemoryError`。

#### 2.3 Stack Area(栈区)
-   每一个线程，都会创建一个单独的 `Runtime Stack(运行时栈区)`。对于每一个方法的调用，都会在栈内存中建立一个条目，称为`堆栈帧`。栈区域不是共享资源，属于每个线程自己的，所以是 `线程安全的`。栈区框架分为三个子实体。
  
    -   局部变量数组
    -   操作数栈区
    -   帧数据
-   如果线程中计算所需的栈超出允许范围，JVM 会抛出 `StackOfOverflowError`。
-   如果可以动态扩展栈区，并尝试扩展，但是没有足够的内存可以实现，JVM 会抛出 `OutOfMemoryError`。
-   如果没有足够的内存可以为新线程创建栈区， JVM 会抛出 `OutOfMemoryError`。

#### 2.4 Program Counter Registers(程序计数寄存器)
-   用来存储指向下一条指令的地址，即将要执行的指令代码，由执行引擎读取下一条指令。

-   程序计数寄存器会保存线程当前的执行指令地址，从而能在多线程并发交替执行时保证切换回来知道从何处继续执行。

-   每个线程都有独立的程序计数寄存器，生命周期和线程的生命周期保持一致。

#### 2.5 Native Method Stack(本地方法栈)
-   本机方法栈保存本机方法信息。对于每个线程，将创建一个单独的本机方法栈。

### 3. Execution Engine(执行引擎)

>   **JVM 执行引擎将高级语言编译为机器语言，保证了不同机器都可以运行。**

#### 3.1 Interpreter(解释器)

- 先使用 `javac` 编译成字节码，然后由 `Interpreter` 进行逐行解释并翻译为本地机器指令，当一条字节码指令执行后，再根据 PC 寄存器记录的下一行被执行字节码执行。
  
  -   在 `HostSpot JVM` 中，解释器由 `Intepreter` 和 `Code` 两个模块构成。
  
      - `Interpreter` 实现解释器核心功能。
      
      - `Code` 管理 `HotSpot JVM` 在运行时生成的本地机器指令。
        
#### 3.2 JIT(Just-in-time Compiler)即时编译器

- 由于解释器相对低效，所以 JVM 支持即时编译技术。即时编译目的是为了避免函数被解释执行，而是将整个函数编译为本地机器指令。每次函数执行时，只需执行编译后的本地机器指令。
  
  -   **JIT 策略**
    
      - **热点代码**
        
          - 一个函数或者函数内的循环体，如果多次被调用，都可以称为 `热点代码`。因此 `JIT` 会编译为本地机器指令。因为这种编译方式发生在函数执行过程，因此被称为 `OSR(On StackReplacement)编译，即栈上替换`。
          
            -   `阈值` 决定函数或循环体调用多少次最终确定为 **热点代码**，从而编译为本地机器指令。
          
      -   **热点探测**
        
        目前 `HotSpot JVM` 采用基于计数器的热点探测，分别为函数创建 **2** 个计数器。
        
        1. **Invocation Counter(调用计数器)**
          -   统计函数被调用次数，默认阈值 `Client 模式 1500次，Server 模式 10000次`。`热点代码` 经过编译后存缓存为 `Code Cache`，存放在元空间。           
          -   `-XX:CompileThreshold=***` 设置
              -   **热度衰减**
                
                  - 调用计数器统计的不是调用绝对次数，而是一个时间范围内的相对执行频率。当超过一定时间限度，调用次数不足以提交给 `JIT` 编译，调用计数器会减半，该过程称为**热度衰减**，该时间段为统计的**半衰周期(Counter Half Life Time)**。
                  
                  -   `-XX:UseCounterDecay` 关闭热度衰减。
                  
                  -   `-XX:CounterHalfLifeTime=***` 设置半衰周期，单位秒。**Debug 版本的 JDK 可以使用该参数**
        
        2. **BackEdge Counter(回边计数器)**
        
            -   统计函数内循环体代码执行次数，在字节码遇到控制流向后跳转的指令称为 `回边(Back Edge)`。回边计数器就是为了触发 `OSR编译`。
        
#### **3.3 Garbage Collection 垃圾回收器**

>   **垃圾回收是 Java 中用来取消分配未使用的内存的机制，除了清除未使用的对象占用的空间外，什么都没有。为了释放未使用的内存，垃圾回收器会跟踪所有仍在使用的对象，并将其余对象标记为垃圾。基本上，垃圾回收器使用 `标记和清除算法` 清除未使用的内存。**
>
>   **编程中的垃圾回收可以通过调用触发 `System.gc()`，但不能保证执行。**

JVM实际上提供了四个不同的垃圾收集器。每个垃圾收集器在 `Application throughput` 和 `Application pause` 上会有所不同。`Application throughput`  表示 Java 应用程序运行的速度，`Application pause` 表示垃圾收集器清理未使用的内存空间所需的时间。

#### 3.3.1 Serial Garbage Collector(串行垃圾收集器)

-   最简单的 GC 实现方式，基本只使用一个线程工作。如果采用该收集器工作，串行垃圾收集器则拥有程序的所有运行线程。

-   **工作原理**

    -   创建一个单线程来执行垃圾收集。`冻结`所有其他正在运行的应用程序线程，直到垃圾收集操作结束。
    
-   **缺点**

    -   造成应用程序吞吐量降低，增加应用程序暂停时间。

#### 3.3.2 **Parallel Garbage Collector(并行垃圾收集器)**

-   `Java 8 默认的垃圾收集器`。

-   设置垃圾收集器在收集时的线程数量

    -   `-XX:+UseParallelGC -XX:ParallelGCThreads=NumberOfThreads -jar GFGApplicationJar.java`
    
-   设置垃圾收集器在收集时的最大暂停时间，单位为毫秒

    -   `-XX:+UseParallelGC -XX:MaxGCPauseMillis=SecInMillisecond -jar GFGApplicationJar.java`
    
-   **工作原理**

    -   并行垃圾收集器与串行工作原理一样。不同的是并行垃圾收集器使用多线程来清理未使用的堆区。
    
-   **存在问题**

    -   在小操作时，依然会暂停应用程序。

#### 3.3.3 Concurrent Mark Sweep(CMS) Collector(并发标记扫除收集器)

-   **工作原理**

    -   使用多个线程持续扫描堆内存，对未使用的对象进行标记，然后对标记的对象进行扫除。
    
-   **冻结2种情况**

    1.  在执行垃圾收集的同时，并行的堆区存发生了变化。
    
    2.  标记了一个在 `Old Generation Space(老年代) `的引用对象。
    
-   **存在问题**

    1.  并发模式故障
    
    2.  过多的 GC 时间和 OutOfMemoryError
    
    3.  内存碎片（浮动垃圾）
    
    4.  GC 周期内多次暂停
    
-   **Oracle 官方声明已弃用。**

    -   [Oracle Docs](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)
    
    -   [Oracle JavaSE HotSpot](https://docs.oracle.com/en/java/javase/11/gctuning/concurrent-mark-sweep-cms-collector.html#GUID-FF8150AC-73D9-4780-91DD-148E63FA1BFF)
    
    -   [OpenJDK  JEP 291](https://openjdk.java.net/jeps/291)
    
    -   [CMS 弃用文章分析](https://www.linkedin.com/pulse/jvm-why-cms-garbage-collector-deprecating-kunal-saxena)

#### 3.3.4 Garbage-First Garbage Collector(G1 垃圾收集器)

-   首先在 JDK 7 中引入了G1垃圾收集器。起初，它的设计是为了更好地支持较大的堆内存应用，G1 Garbage Collector 是 `Java 9的默认垃圾收集器`。G1收集器取代了CMS收集器，因为它的性能更高效。

-   Java 9 版本以下启动 G1 命令

    -   `XX：+ UseG1GC -jar GFGApplicationJar.java`
    
-   **工作原理**

    -   G1收集器将堆空间分割成多个大小相等的区域。基本上，它主要是为堆大于 `4GB` 的应用程序设计的。它将堆空间划分为多个区域，从 1MB 到 32MB 不等。
    
    -   G1收集器将堆划分为若干个区域（Region），它仍然属于分代收集器。不过，这些区域的一部分包含新生代，新生代的垃圾收集依然采用暂停所有应用线程的方式，将存活对象拷贝到老年代或者 Survivor 空间。老年代也分成很多区域，G1收集器通过将对象从一个区域`复制`到另外一个区域，完成了`清理`工作。这就意味着，在正常的处理过程中，G1完成了`堆的压缩（至少是部分堆的压缩）`，这样也就不会有 CMS 内存碎片问题的存在了。

### 4. JNI(Native Method Interface/本地接口)

-   JNI 与本机方法库进行交互，并提供执行引擎所需的本机方法库。

### 5. Native Method Library(本地方法库)

-   本机方法库的集合，执行引擎所必须的。
