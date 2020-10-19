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

![JVM-Architecture](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/JVM-Architecture.png)

  - `JVM` 是 `JRE` 的实例，存在于内存中并与内存打交道，是 Java 语言实现跨平台的核心，即 `Write once, run anywhere!` 
  - **工作原理**
    
    - `JVM` 联合 `JRE`  的 utils、集合框架等，执行 `Java编译器` 编译后的 `*.class` 字节码文件，解释字节码到 `CPU指令集` 或 `OS系统` 调用。
    
#### ClassLoader SubSystem(类加载器子系统)

> **Java 的动态类加载由该系统负责处理。类加载子系统会加载、链接，并在运行时首次引用类的初始化类文件。该系统一共分为三个步骤。**

##### 1. Loading(加载)

**类的字节码文件将由此组件加载。BootStrap ClassLoader，Extension ClassLoader 和 Application ClassLoader 是有助于实现该目标的三个 ClassLoader。**

1. **BootStrap ClassLoader**
    -   负责从引导类路径中加载类，仅用于 `rt.jar`。最高优先级将给予此加载程序。

2. **Extension ClassLoader**
    -   负责加载 `ext文件夹(jre\lib)`内的类。
  
3. **Application ClassLoader**
    -   负责加载应用程序级别的类路径、环境变量路径等。

##### 2. Linking(链接)

1.  **Verify(验证)**
    -   字节码验证程序验证生成的字节码是否正确，如果验证失败，将会收到验证错误。
2.  **Prepare(准备)**
    -  为类的`static(静态变量)`分配内存，并设置默认值。
3.  **Resolve(解析)**
    -  将所有符号存储引用替换为 `Method Area` 的原始引用。

##### 3. Initialization(初始化)

1.  这是 ClassLoading 的最后阶段。在此，所有 `static(静态变量)`将被分配默认值，并且将执行 `静态块`。

#### Runtime Data Areas(运行时数据区)

>   **运行时数据区分为五个部分。**

1.  **Method Area(方法区域)**
    1.  所有类级别的数据**（包括静态变量）**都将存储在此区域。每个 JVM 实例都只有一个方法区域，JVM 在启动时创建该区域，为线程间共享。该区域的内存不必是连续的，且可以指定方法区域的大小。
    2.  如果无法提供方法区域的内存来满足分配需求，JVM 会抛出 `OutOfMemoryError`。
2.  **Heap Area(堆区)**
    1.  所有对象及其对应的实例变量和数组都将存储在此区域。每个 JVM 实例还有一个 `Heap Area(堆区)`。由于 `Method Area` 和 `Heap Area` 为多个线程共享内存，所以`存储的数据不是线程安全的`。
    2.  如果计算需要的堆多余自动存储管理系统可以提供的堆，JVM 会抛出 `OutOfMemoryError`。
3.  **Stack Area(栈区)**
    1.  每一个线程，都会创建一个单独的 `Runtime Stack(运行时栈区)`。对于每一个方法的调用，都会在栈内存中建立一个条目，称为`堆栈帧`。栈区域不是共享资源，属于每个线程自己的，所以是 `线程安全的`。栈区框架分为三个子实体。
        1.  局部变量数组
        2.  操作数栈区
        3.  帧数据
4.  **PC Registers(PC 寄存器)**
5.  **Native Method Stack(本地方法栈)**

#### Eexcution Engine(执行引擎)

>   **JVM 执行引擎将高级语言编译为机器语言，保证了不同机器都可以运行。**

##### 1. Interpreter(解释器)
- 先使用 `javac` 编译成字节码，然后由 `Interpreter` 进行逐行解释并翻译为本地机器指令，当一条字节码指令执行后，再根据 PC 寄存器记录的下一行被执行字节码执行。
  
  -   在 `HostSpot JVM` 中，解释器由 `Intepreter` 和 `Code` 两个模块构成。
      - `Interpreter` 实现解释器核心功能。
        - `Code` 管理 `HotSpot JVM` 在运行时生成的本地机器指令。
##### 2. JIT(Just-in-time Compiler)即时编译器
- 由于解释器相对低效，所以 JVM 支持即时编译技术。即时编译目的是为了避免函数被解释执行，而是将整个函数编译为本地机器指令。每次函数执行时，只需执行编译后的本地机器指令。
  
  -   **JIT 策略**
      
      - **热点代码**
        
          - 一个函数或者函数内的循环体，如果多次被调用，都可以称为 `热点代码`。因此 `JIT` 会编译为本地机器指令。因为这种编译方式发生在函数执行过程，因此被称为 `OSR(On StackReplacement)编译，即栈上替换`。
            -   `阈值` 决定函数或循环体调用多少次最终确定为 **热点代码**，从而编译为本地机器指令。
          
      -   **热点探测**
        
        目前 `HotSpot JVM` 采用基于计数器的热点探测，分别为函数创建 **2** 个计数器。
        
        - **1. Invocation Counter(调用计数器)**
          
          -   统计函数被调用次数，默认阈值 `Client 模式 1500次，Server 模式 10000次`。`热点代码` 经过编译后存缓存为 `Code Cache`，存放在元空间。           
          -   `-XX:CompileThreshold=***` 设置
              -   **热度衰减**
                  
                  - 调用计数器统计的不是调用绝对次数，而是一个时间范围内的相对执行频率。当超过一定时间限度，调用次数不足以提交给 `JIT` 编译，调用计数器会减半，该过程称为**热度衰减**，该时间段为统计的**半衰周期(Counter Half Life Time)**。
                  
                  -   `-XX:UseCounterDecay` 关闭热度衰减。
                  -   `-XX:CounterHalfLifeTime=***` 设置半衰周期，单位秒。**Debug 版本的 JDK 可以使用该参数*
          
        - **2. BackEdge Counter(回边计数器)**
          
             -   统计函数内循环体代码执行次数，在字节码遇到控制流向后跳转的指令称为 `回边(Back Edge)`。回边计数器就是为了触发 `OSR编译`。
##### **3. Garbage Collection 垃圾回收器**

>   **收集并删除未引用的对象。垃圾回收可以通过调用触发 `System.gc()`，但不能保证执行。**






