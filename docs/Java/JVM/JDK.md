# **Java** 概述

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

- **类加载器主要遵循 4 个主要原则**
  1. **Visibility Principle(可见性原则)**
     - 该原则规定，子类加载器可以看到父类加载器所加载的类，但父类加载器不能找到子类加载器所加载的类。
  2. **Uniqueness Principle(唯一性原则)**
     - 该原则规定，被父类加载的类不应再被子类加载器加载，确保不发生重复加载类的情况。
  3. **Delegation Hierarchy Principle(委托层次原则)**
     - 为了满足以上2个原则，JVM 遵循授权的层次结构来选择每个类加载请求的类加载器。这里，从最低的子级开始，`Application ClassLoader` 将收到的类加载请求委托给 `Extension ClassLoader`，然后 `Extension ClassLoader`再将请求委托给 `Bootstrap ClassLoader`。如果请求的类在 `Bootstrap` 的路径中找到，则该类被加载，否则请求再次转回 `Extension ClassLoader`。从 `Extension` 路径或自定义指定的路径中找到类。如果它也失败了，请求就会回到 `Application ClassLoader`，从 `System` 类路径中找到类，如果 `Application ClassLoader` 也不能加载所请求的类，那么我们就会得到运行时异常 `java.lang.ClassNotFoundException`。
  4. **No Unloading Principle(无卸载原则)**
     - 尽管类加载器可以加载一个类，但它不能卸载已加载的类，而可以删除当前的类加载器，并创建新的类加载器。除了卸载，还可以删除当前的类加载器，并创建一个新的类加载器。

#### 1.1 Loading(加载)

**类的字节码文件将由此组件加载。BootStrap ClassLoader，Extension ClassLoader 和 Application ClassLoader 是有助于实现该目标的三个 ClassLoader。**

- **BootStrap ClassLoader**

  -   负责从 `rt.jar` 加载标准 `JDK 类`。最高优先级将给予此加载程序。
      -   例如引导路径中存在的核心 Java API类 `$JAVA_HOME/jre/lib`目录（例如: java.lang.*）。它以C / C ++之类的本地语言实现，并充当 Java 中所有类加载器的父级。

- **Extension ClassLoader**

  -   负责加载 `ext文件夹(jre/lib/ext)`内的类。
      -   将类加载请求委托给其父类 `Bootstrap`，如果不成功，则从扩展路径中的扩展目录（例如，安全扩展函数）（`$JAVA_HOME/jre/lib/ext` 或 `java.ext` 指定的任何其他目录）中加载类。 `.dirs` 系统属性。该类加载器由 `java.sun.misc.Launcher$ExtClassLoader` 类实现。

-   **Application ClassLoader**

    -   负责加载应用程序级别的类路径、环境变量路径等。
        -   从系统类路径加载应用程序特定的类，可以在调用程序时使用 `-cp` 或 `-classpath` 命令行选项进行设置。它在内部使用环境变量映射到 `java.class.path`。这个类加载器在 Java 中由`sun.misc.Launcher$AppClassLoader` 类实现。
    
-   除了上面讨论的 3 个主要的类加载器外.还可以直接在代码本身上创建一个用户定义的类加载器。这样通过**类加载器授权模式**，保证了应用程序的独立性。这种方法在 `Tomcat` 等 We b应用服务器中使用，使 Web 应用和企业解决方案独立运行。

    每个 `ClassLoader`都有它的命名空间，存储加载的类。当 `ClassLoader` 加载一个类时，它会根据存储在命名空间中的 **FQCN（Fully Qualified Class Name，完全合格的类名）** 搜索该类，检查该类是否已经被加载。即使类的 **FQCN** 相同，但名称空间不同，也会被视为不同的类。**不同的命名空间意味着该类已经被另一个类加载器加载。**

#### 1.2 Linking(链接)

##### 1.2.1. Verify(验证)

-   字节码验证程序验证生成的字节码是否正确，如果验证失败，将会收到验证错误 `java.lang.VerifyError`。
    -   检查 `.class` 文件
        -   格式一致的符号表
        - `final` 修饰的方法、类不被覆盖
        - 方法是否遵循访问控制关键字
        - 方法有正确的参数数量和类型
        - 字节码不会错误的操作堆栈
        - 变量在被读取前已初始化
        - 变量的值是否是一个正确的类型

##### 1.2.2 Prepare(准备)

-  为类的`static(静态变量)`分配内存，并设置**默认值**，但是在此阶段不执行任何初始化程序或代码。

##### 1.2.3  Resolve(解析)

-  将所有符号存储引用替换为 `Method Area` 的原始引用，通过搜索 `Method Area` 以找到引用的实体来完成此操作。

#### 1.3 Initialization(初始化)

-   这是 `ClassLoading` 的最后阶段。在此，所有 `static(静态变量)`将被分配**原始值（代码中定义的值）**，并且将执行 `静态块`。
-   在这里，每个加载的类或接口的初始化逻辑将被执行**（例如调用类的构造函数）**。由于 JVM 是多线程的，一个类或接口的初始化应该非常小心地发生，并有适当的 `synchronization`，以避免其他线程试图同时初始化同一个类或接口（即使其线程安全）。

`以上 1.1 - 1.3 共 5 个步骤，属于类加载过程。`

### 2. Runtime Data Areas(运行时数据区)

>   **运行时数据区分为五个部分。**

#### 2.1 Method Area(方法区域 -- 线程间共享)

-   每个 JVM 实例都只有一个方法区域，**JVM 启动时创建该区域**，为线程间共享。
-   该区域的内存不必是连续的，且可以指定方法区域的大小。JVM 规范不强制在堆中实现方法区域。例如，在 **Java 7**之前，Oracle **HotSpot** 使用一个名为 `PermGen` 的区域来存储 `方法区域`。该 `PermGen` 与 Java 堆（以及由 JVM 像堆一样由 JVM 管理的内存）是连续的，默认大小在 32 位 JVM 上为 64 MB，在 64 位版本上为 82 MB。`PermGen` 在 **Java 8** 中已被完全删除。代替 PermGen 的是，引入了一个名为 `Metaspace` 的新功能。默认情况下，`Metaspace` 自动增长，最大可用空间是总可用系统内存。在此，当类元数据使用量达到其最大 `Metaspace` 大小时，将**自动触发垃圾收集**。
-   **Metaspace 与 Perm Gen 之间最大的区别在于：Metaspace 并不在虚拟机中，而是使用本地内存。**
-   **方法区域存储以下三种信息**
    1. 类信息（字段、方法的数量，超类名称，接口名称，版本等）。
    2. 方法和构造函数的字节码。
    3. 每个类加载的运行时常量池。
-   **异常情况**
    -   如果无法提供方法区域的内存来满足分配需求，JVM 将抛出 `OutOfMemoryError`。

##### 2.1.1 Runtime constant pool(运行时常量池)

- 该池是`方法区域`的子部分。由于它是元数据的重要组成部分，因此 **Oracle** 规范描述`方法区域`之外的运行时常量池。对于每个加载的类、接口，此常量池都会增加。该池就像常规编程语言的符号表。当引用类，方法或字段时，JVM 使用运行时常量池在内存中搜索实际地址。它还包含常量值，例如字符串文字或常量图元。

#### 2.2 Heap Area(堆区 -- 线程间共享 -- 垃圾收集器工作区域)
-   `堆区代表运行时数据区`，从中为所有`类实例`和`数组`分配内存，所有对象及其对应的实例变量和数组都将存储在此区域。并在 **JVM 启动时创建该区域**。每个 JVM 实例还有一个 `Heap Area(堆区)`。由于 `Method Area` 和 `Heap Area` 为多个线程共享内存，所以`存储的数据不是线程安全的`。
-   堆可以是固定大小，也可以是动态大小（基于系统的配置），并且分配给堆区域的内存不必是连续的。
-   堆是所有 JVM 线程之间共享的内存区域，因为 Java 无需开发人员分配内存，所有**堆区域必须由垃圾收集器来管理。**
-   **异常情况**
    -   如果计算需要的堆多余自动存储管理系统可以提供的堆，JVM 将抛出 `OutOfMemoryError`。

#### 2.3 Stack Area(栈区 -- 单个线程独享)
-   每一个线程，都会创建一个单独的 `Runtime Stack(运行时栈区)`。对于每一个方法的调用，都会在栈内存中建立一个条目，称为`堆栈帧`。栈区域不是共享资源，属于每个线程自己的，所以是 `线程安全的`。每次调用方法时，都会创建一个新 `Frame` 并将其放入栈中。框架的方法调用完成后，无论该完成是正常的还是异常的（它引发未捕获的异常），都会被销毁。

-   栈区可以是固定大小，也可以是动态大小。如果具有固定大小，则在创建该堆栈时可以独立选择每个堆栈的大小。

-   给定线程中的任何一点都只有一个 `Frame（用于执行方法的框架）`处于活动状态。该帧称为**当前帧**，其方法称为**当前方法**。定义当前方法的**类**是**当前类**。局部变量和操作数堆栈上的操作通常参考当前帧。

-   **栈区 Frames 是一种数据结构，分为三个子实体。**
  
  ![Stack-Frame](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Stack-Frame.png)
  
  -   **Local variable array(局部变量数组)**
        -   此数组包含当前方法范围内的所有局部变量。该数组可以保存基本类型，引用或 returnAddress 的值。该数组的大小是在编译时计算的。JVM 使用局部变量在方法调用时传递参数，被调用方法的数组是从调用方法的操作数堆栈中创建的。
    -   **Operand Stack(操作数栈区)**
        -   这个栈被字节码指令用来处理参数。该栈还用于在（Java）方法调用中传递参数，并在调用方法的栈顶部获取被调用方法的结果。
    -   **Run-time constant pool reference(运行时常量池引用)**
        -   引用正在执行的**当前方法**的**当前类**的常量池。JVM 使用它来将符号方法、变量引用（例如: myInstance.method()）转换为实际的内存引用。
  
-   **异常情况**
    -   如果线程中计算所需的栈超出允许范围，JVM 将抛出 `StackOfOverflowError`。
    -   如果可以动态扩展栈区，并尝试扩展，但是没有足够的内存可以实现，JVM 将抛出 `OutOfMemoryError`。
    -   如果没有足够的内存可以为新线程创建栈区，JVM 将抛出 `OutOfMemoryError`。

#### 2.4 Program Counter Registers(程序计数寄存器 -- 单个线程独享)
-   **每个线程都有独立的程序计数寄存器，生命周期和线程的生命周期保持一致。**
-   用来存储指向下一条指令的地址，即将要执行的指令代码，由执行引擎读取下一条指令。
-   **程序计数寄存器会保存线程当前的执行指令地址，从而能在多线程并发交替执行时保证切换回来知道从何处继续执行。**

#### 2.5 Native Method Stack(本地方法栈 --  单个线程独享)
-   本地方法栈也称为 `C栈`，保存本地方法信息，通常在`创建每个线程时为每个线程分配`。**Java线程和本机操作系统线程之间存在直接映射**。在为 Java 线程准备好所有状态之后，还将创建一个单独的本机堆栈，以存储通过`JNI（Java本机接口）`调用的本机方法信息（通常用C / C ++编写）。创建并初始化本地线程后，它将调用 Java 线程中的 `run()` 方法。当 `run()` 方法返回时，将处理未捕获的异常（如果有），然后本地线程确认是否需要由于线程终止而终止 JVM（即，它是最后一个非守护线程）。当线程终止时，将释放本地线程和 Java 线程的所有资源。Java 线程终止后，将回收本地线程。因此，操作系统负责调度所有线程并将其分配给任何可用的CPU。
-   本地方法堆栈的大小可以是固定的，也可以是动态的。
-   **异常情况**
    -   如果线程中的计算所需的本地方法堆栈超出允许的范围，JVM 将抛出`StackOverflowError`。
    -   如果本地方法堆栈可以动态扩展，并且尝试扩展本机方法堆栈，但可用内存不足。或者可用内存不足，无法为新线程创建初始本地方法堆栈，JVM 将抛出`OutOfMemoryError`。

### 3. Execution Engine(执行引擎)

>   **JVM 执行引擎将高级语言编译为机器语言，保证了不同机器都可以运行。字节码的实际执行在这里进行。执行引擎通过读取分配给上述运行时数据区域的数据逐行执行字节码中的指令。**

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
        
#### 3.3 Garbage Collection(垃圾回收器) 

>   **垃圾回收是 Java 中用来取消分配未使用的内存的机制，除了清除未使用的对象占用的空间外，什么都没有。为了释放未使用的内存，垃圾回收器会跟踪所有仍在使用的对象，并将其余对象标记为垃圾。基本上，垃圾回收器使用 `标记和清除算法` 清除未使用的内存。**
>
>   **编程中的垃圾回收可以通过调用触发 `System.gc()`，但不能保证执行。**

JVM 实际上提供了四个不同的垃圾收集器。每个垃圾收集器在 `Application throughput` 和 `Application pause` 两个指标上衡量会有所不同。`Application throughput`  表示 Java 应用程序运行的速度，`Application pause` 表示垃圾收集器清理未使用的内存空间所需的时间。

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
    
    2.  过多的 GC 时间和 `OutOfMemoryError`
    
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

-   JNI 与本地方法库进行交互，并提供执行引擎所需的本机方法库功能（通常用C / C ++编写）。JNI 使得 JVM 可以调用 C/C++ 库。

### 5. Native Method Library(本地方法库)

-   本地方法库的集合，执行引擎所必须的。
