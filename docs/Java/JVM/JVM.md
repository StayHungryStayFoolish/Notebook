# JVM 内存模型

![JRE](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Java-Runtime-data.png)

> 此文只讨论 JVM 内存模型，如需了解上图全部结构，请查看 [JDK 三大组件](http://notebook.bonismo.ink/#/Java/JVM/JDK)。

- JVM 是使计算机能够运行 Java 程序的抽象计算机。JVM 有三种概念： 
  - **规范** （指定 JVM 的工作方式。但是实现由Sun和其他公司提供）
  - **实现** （称为（JRE）Java Runtime Environment）
  - **实例** （在编写 Java 命令之后运行） Java类，将创建 JVM 的实例）。

- Java虚拟机加载代码，验证代码，执行代码，管理内存（这包括从操作系统（OS）分配内存，管理 Java 分配，包括堆压缩和垃圾对象的删除），并最终提供运行时环境。

## JVM 内存结构

### JVM 在 OS 内存结构

![HostMemory](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/HostMemory.png)

### JVM 内部内存结构

![jvm_memory_structure](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/jvm_memory_structure.gif)

> **在 JVM 内部，存在单独的内存空间（堆，非堆，缓存），以便存储运行时数据和编译后的代码。**

#### 1. Heap Memory(堆内存)

![HeapMemory](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/HeapMemory.png)

> **JVM 堆内存采用分代模型，分为两大部分：年轻代、老年代。其中年轻代又分为三个区域：Eden Memory(伊甸园区)、Survivor Memory(幸存者区S0、S1)**

- JVM 启动时使用 `-Xms` 指定初始大小、`-Xmx` 指定最大大小。

##### 1.1 Young Gen(年轻代)

- **Java 对象在年轻代的生命周期**
  1. 保存新分配的对象。大多数新创建的对象都被分配在 `Eden 区`。当 `Eden 区` 被创建的对象填满时，将执行 `Minor GC` ，并将幸存者对象移动到其中一个 `Survivor 区`。
  2. `Minor GC` 也会检查幸存者对象，并将它们移动到另一个 `Survivor 区`。所以每次都会有一个 `Survivor 区` 是空的。
  3. 经过多次 `GC` 循环后仍然存在的对象将移至 `老年代` 存储空间。通常，可以通过设置年轻对象的年龄阈值，然后再有资格晋升为老对象。

##### 1.2 Old Gen(老年代)

- **Java 对象在老年代的生命周期**
  - 这是为包含可能在多轮次要GC中存活下来的长寿命对象而保留的
  - 当旧一代空间已满时，将执行 `Major GC`（通常需要更长的时间）

#### 2. Non-Heap Memory(非堆内存)

![Non-Heap Memory](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Non-Heap Memory.png)

##### 2.1 Metaspace

- 上图的 `Perm Gen` 为 **Java 7** 的永久代。在 **Java 8** 已由 `Metaspace` 替换。默认情况下，`Metaspace` 自动增长，最大可用空间是总可用系统内存。在此，当类元数据使用量达到其最大 `Metaspace` 大小时，将**自动触发垃圾收集**。
- **Metaspace 与 Perm Gen 之间最大的区别在于：Metaspace 并不在虚拟机中，而是使用本地内存。**
- `Metaspace` 对应的 `Runtime Data Area` 的 `Method Area`，分别存储以下三种信息
  1. 类信息（字段、方法的数量，超类名称，接口名称，版本等）。
  2. 方法和构造函数的字节码。
  3. 每个类加载的运行时常量池。

#### 3. Other Memory(其他内存)

##### 3.1 其他内存主要用于 Cache，包括三部分

1. 这包括**代码缓存**
2. 存储由JIT编译器生成的已编译代码（即本地代码），JVM 内部结构，已加载的 `profiler` 代码和数据等。
3. 当代码缓存超过阈值时，它将被刷新（并且GC不会重新定位对象）。

### 堆、非堆与栈的关系

![Heap-Non-Stack](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Stack&Heap.png)

