# JVM 类加载机制

## 概述

### Java 三大组件 JDK、JRE、JVM

- `JDK(Java Development Kit)` 
  - `JDK`是开发 Java 引用程序的软件开发环境。`JDK` 包括 `JRE(Java Runtime Environment)`运行时环境、`Java`启动器(解释器/加载器)、`javac`(编译器)、`jar`(归档器)、`javadoc`(文档生成器)以及开发所需其他工具。
    - JDK 面向的是开发人员，如果只需运行程序，可以单独下载 JRE。
- `JRE(Java Runtime Environment)`
  - `JRE`是构建了一个可以在其中执行 Java 程序的运行时环境。`JRE`提供了执行 Java 应用程序的最低要求。`JRE` 包括 Java 语言常用的 utils、集合框架等核心类库、`JVM(Java Virtual Machine)虚拟机`等。
  - **工作原理**
    - `JRE` 是 `JDK` 的一部分，在磁盘上。`JRE` 使用 Java 字节码(javac 执行后产生)，将其所需要的 Library 组合，然后启动 `JVM` 来执行。
- `JVM(Java Virtual Machine)`
  - `JVM` 是 `JRE` 的实例，存在于内存中并与内存打交道，是 Java 语言实现跨平台的核心，即 `Write once, run anywhere!` 
  - **工作原理**
    - `JVM` 联合 `JRE`  的 utils、集合框架等，执行 `Java编译器` 编译后的 `*.class` 字节码文件，解释字节码到 `CPU指令集` 或 `OS系统` 调用。
  - **`JVM ` 执行引擎**
    - **JVM 执行引擎将高级语言编译为机器语言，保证了不同机器都可以运行。**
    - **1. Interpreter(解释器)**
      - 先使用 `javac` 编译成字节码，然后由 `Interpreter` 进行逐行解释并翻译为本地机器指令，当一条字节码指令执行后，再根据 PC 寄存器记录的下一行被执行字节码执行。
      - 在 `HostSpot VM` 中，解释器由 `Intepreter` 和 `Code` 两个模块构成。
        - `Interpreter` 实现解释器核心功能。
        - `Code` 管理 `HotSpot VM` 在运行时生成的本地机器指令。
    - **2. JIT(Just-in-time Compiler)即时编译器**
      - 由于解释器相对低效，所以支持即时编译技术。即时编译目的为了避免函数被解释执行，而是将整个函数编译为本地机器指令。每次函数执行时，只需执行编译后的本地机器指令。
      - **JIT 策略**
        - **热点代码**
          - 一个函数或者函数内的循环体，如果多次被调用，都可以称为 `热点代码`。因此 `JIT` 会编译为本地机器指令。因为这种编译方式发生在函数执行过程，因此被称为 `OSR(On StackReplacement)编译，即栈上替换`。
          - `阈值` 决定函数或循环体调用多少次最终确定为 **热点代码**，从而编译为本地机器指令。
          - 目前 `HotSpot VM` 采用基于计数器的热点探测。
        - **热点探测**
          - 方法调用计数器
            - 统计方法被调用次数，默认阈值 `Client 模式 1500次，Server 模式 10000次`。`热点代码` 经过编译后存缓存为 `Code Cache`，存放在元空间。
              - `-XX:CompileThreshold=****` 设置
          - 回边计数器
    - **3. Garbage Collection 垃圾回收期**





