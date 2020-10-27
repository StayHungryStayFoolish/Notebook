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

>   **Young Gen 分为 3 个区，Eden 占用 80%，两个 Survivor 分别占用 10%。两个 Survivor 简称 S0, S1，在有些地方也称 From，To (JVM 打印日志会以该名称显示)。**

- **Java 对象在年轻代的生命周期（复制算法实现步骤）**

  1. 保存新分配的对象。大多数新创建的对象都被分配在 `Eden 区`。当 `Eden 区` 被创建的对象填满时，将执行 `Minor GC` ，并将幸存者对象一次性移动到其中一个 `Survivor 区`。
  
  2. `Minor GC` 也会检查幸存者对象，并将它们移动到另一个 `Survivor 区`。所以每次都会保证有一个 `Survivor 区` 是空的，**即 90% 的内存都可以被使用。**
  
  3. `Eden 区`的 对象将移至 `Old Gen` ，有 **4** 种情况。
  
      1. **GC 次数判断**
      
          -   通常，可以通过设置年轻对象的年龄阈值，然后再有资格晋升为老对象，**默认为 15 次，参数 `-XX:MaxTenuringThreshold` 可以设置。**
          
      2. **动态对象年龄判断**
      
          -   一批对象总大小**大于**当前 `Survivor区` 内存的大小的 **50%**，那么大于等于这批对象年龄的对象就会被转移到老年代。**每一次GC，年龄+1。如果 年龄1 + 年龄2 + 年龄N > 50%，则年龄N的会移到 Old Gen。**
          
      3. **大对象直接进入 Old Gen**
      
          -   参数 `-XX:PretenureSizeThreshold` 可以设置对象大小，默认为 **0**。如果设置该阈值，则超过的不会分配到 `Eden 区` 而是直接进入 `Old Gen`。
          
      4. **Minor GC 后存活的对象太多，无法进入 Survivor 区**
      
          -   `Survivor 区` 无法存储，则会直接存储到 `Old Gen`。

##### 1.2 Old Gen(老年代，采用标记扫描整理垃圾)

**注意：JVM 规范或 Garbage Collection 研究论文中没有这些术语的正式定义。**

- **Java 对象在老年代的生命周期（标记整理算法实现步骤）**

  - 这是为包含可能在多轮 `Minor GC` 中存活下来的长寿命对象而保留的，一般 `Young Gen` 经过 **15** 次 `Minor GC` 后存活下来，会转入老年代。
  
  - 当 `Old Gen` 空间已满时，将执行 `Major GC`（通常需要更长的时间）

#### 2. Non-Heap Memory(非堆内存)

![Non-Heap Memory](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Non-Heap Memory.png)

##### 2.1 Metaspace

- 上图的 `Perm Gen` 为 **Java 7** 的永久代。在 **Java 8** 已由 `Metaspace` 替换。默认情况下，`Metaspace` 自动增长，最大可用空间是总可用系统内存。在此，当类元数据使用量达到其最大 `Metaspace` 大小时，将**自动触发垃圾收集**。

- **Metaspace 与 Perm Gen 之间最大的区别在于：Metaspace 并不在虚拟机中，而是使用本地内存。**

- `Metaspace` 包含 `Runtime Data Area` 的 `Method Area`，分别存储以下三种信息

  1. 类信息（字段、方法的数量，超类名称，接口名称，版本等）。
  
  2. 方法和构造函数的字节码。
  
  3. 每个类加载的运行时常量池。

#### 3. Other Memory(其他内存)

##### 3.1 其他内存主要用于 Cache，包括三部分

1. **代码缓存**

2. 存储由 **JIT** 编译器生成的已编译代码（即本地代码），JVM 内部结构，加载的 `profiler` 组件的代码和数据等。

3. 当代码缓存超过阈值时，将被刷新（并且 `GC` 不会重新定位对象）。

### 堆、非堆与栈的关系

![Heap-Non-Stack](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Stack&Heap.png)

- **注：Non Heap 图中的 Permanent Generation 在 Java 8 中已更换为 Metaspace**

| Parameter | Stack Memory                                                 | Heap Memory                                                  |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 应用      | **1. 分次使用**（只在线程执行中使用）**2. 线程安全**（每个线程只在自己的栈中运行，具体可以参考图中的 `Frame`） | **1. 共享资源**（整个应用程序在运行期间共享堆内存空间）**2. 线程不安全**（需要使用同步代码保护资源） |
| 大小      | 大小取决于操作系统，通常小于堆                               | 用户自定义或默认，没有操作系统的大小限制。                   |
| 存储      | 仅`局部变量`和在堆中创建的`对象的引用`                       | 所有新创建的`类对象`、`数组`                                 |
| 访问顺序  | 使用后进先出（LIFO）内存分配技术进行访问                     | 通过复杂的内存分代管理技术进行访问，包括年轻代，老年代、元空间 |
| 生命周期  | 只要当前方法正在运行，就会存在                               | 只要应用程序运行就会存在                                     |
| 分配效率  | 与堆相比，分配速度相对快很多。                               | 与栈相比，分配速度较慢                                       |
| 分配/释放 | 当一个方法被调用和返回时，这个内存会自动分配和重新分配。     | 当新的对象被创建时，堆空间被分配，当它们不再被引用时，Gargabe Collector会回收。 |

#### 简单分析代码在堆栈的存储

```Java
class Person {
    int pid;
    String name;
    // constructor, setters/getters
}
public class Test {
    public static void main(String[] args) {
        int id = 23;
        String pName = "Jon";
        Person p = null;
        p = new Person(id, pName);
    }
}
```

![Code-Heap-Stack](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/0_HWkfCG1q4DFsFfoF.jpeg)

- 观察上图可以发现，`int` 类型的 **id = 23** 是在当前 `Stack` 内的 `Frame` 中，**p** 是 `Heap` 内 Person 对象的引用，**pName** 因为是 `String` 类型，所以该引用也指向 `Heap` 内的 `String Pool`。

##### String Pool

- Java 中 String 的定义为 `final`。所以 JVM 内部采用了 `Flyweight` 的设计模式，为了能在 Java 运行时节省大量空间，通过`引用指向`的方式，采用了字符串池存储。

- 注：上述 `String Pool` 只会存储 `String s = "value"`，如果采用 `new String` 方式则会按照创建对象方式存储到 `Heap` 中。

  - ```java
    String s1 = "abc"
    String s2 = "abc"
    s1 == s2 // True，因为s1,s2 引用都指向 String Pool 中的 value，
    
    String s3 = "abc";
    String s4 = new String("abc");
    s3 == s4 // False
    
    String s5 = new String("abc");
    String s6 = new String("abc");
    s5 == s6 // False
      
    String s7 = "abc";
    String s8 = "ab" + "c"; // 该方式属于字符串常量表达式
    s7 == s8 // True
      
    String.intern() 表示首先会去 String Pool 中查找当前字符串，如果没有会将该字符串放入 String Pool
      
    String s9 = "abc";
    String s10 = "ab";
    String s11 = s9 + "c";
    s9 == s11 // False
    s9 == s11.intern() // True  
    ```

### JVM 优化厂商

-   **Oracle 的 JIT 编译器模型 Hotspot**

    -   `Hotspot` 分为 **Client** 和 **Server** 模式，使用 `java -version` 可以看到本机模式。**Client** 旨在通过减少应用程序启动时间和内存占用量，主要运行在客户端环境。**Server** 旨在提高最大峰值的运行速度，占用更多的内存，**目前 64 位处理器默认使用 Server 模式**。主要区别在内部的**编译器级别**。
    -   Client 编译器不会尝试执行由服务器VM中的编译器执行的许多更复杂的优化，但是作为交换，它需要较少的时间来分析和编译一段代码。这意味着客户端VM可以更快地启动，并且需要较小的内存空间。
        -   Server 包含一个高级自适应编译器，该编译器支持通过优化 C++ 编译器执行的许多相同类型的优化，以及一些传统编译器无法完成的优化，例如跨虚拟方法调用的主动内联。与静态编译器相比，这是一个竞争优势和性能优势。自适应优化技术的方法非常灵活，通常甚至优于高级静态分析和编译技术。
-   **Oracle 收购 BEA 后的 JRockit**
    -   在 JDK8 中已将部分功能融合如 Hotspot。
-   **IBM 的 AOT(Ahead-Of-Time) 编译器模型 J9**

    -   J9 的模块化程度更高，但因为许可协议限制，只能在 IBM 的产品上使用。
    -   JVM 共享通过共享缓存编译的本机代码，因此已经通过 AOT 编译器编译的代码可以由另一个 JVM 使用，而无需编译。此外，IBM JVM通过使用 AOT 编译器将代码预编译为 JXE（Java可执行文件）文件格式，提供了一种快速的执行方式。

## 垃圾回收机制

> **许多人认为垃圾收集会收集并清理不再存活的对象。实际上，Java 垃圾收集是完全相反的，Java 通过跟踪活动对象，其他的一切都会被指定为垃圾。**

### Java Garbage Collection Roots(GC Roots / Java 垃圾收集根)

Java 内存管理及其内置的垃圾回收，是该语言最出色的成就之一。它可以使开发人员可以创建对象，而无需担心内存的分配与释放，因为垃圾收集器会自动回收内存以供重用。内存管理与垃圾回收机制使得编程更加快捷，同时也避免了内存泄漏和其他内存问题（理论上）。

Java 垃圾回收似乎工作得很好，创建和删除了太多对象。可以解决大多数内存管理问题，但通常以造成严重的性能问题为代价。使垃圾收集适应各种情况已经导致了一个复杂且难以优化的系统。

JVM 的 `Heap Memory` 主要用于动态分配内存，`OS` 会在程序运行时预先分配 `Heap` 交由 JVM 来管理。主要有两方面的影响：

1. 对象创建速度更快，因为对象不需要与 `OS` 全局同步。分配只申请了内存数组的一部分，然后将偏移指针向前移动。下一次分配从该偏移量开始，并申请内存数组下一部分。

2. 当一个对象不再使用时，垃圾收集器会回收底层内存，并将其重新用于分配对象。这意味着没有显式删除，也没有将内存还给 `OS`。

![Memory-Layout](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Memroy-Layou.jpeg)

>   **新对象仅在已使用堆得末尾分配。**

所有的对象都分配在 JVM 管理的堆区。只要一个对象被引用，JVM 就认为它是活着的。一旦一个对象不再被引用，因此不能被应用程序代码到达，垃圾回收器就会将其删除，并回收未使用的内存。听起来很简单，但却带来了一个问题：**树中的第一个引用是什么？**

**每个对象树必须具有一个或多个根对象。只要应用程序可以到达这些根，则整个树都可以到达。有一些特殊的对象称为`垃圾回收根（GC根）`。`它们充当垃圾收集标记机制的根对象。`**

![GC-Roots](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/GC-Roots.png)

-   **Java 中有四种 GC根**
    -   **Stack 内当前方法的局部变量和参数**
        -   通过线程栈保持活跃状态。这不是一个真正的对象虚拟引用，因此不可见。就意图和目的而言，局部变量是 GC 的根。
    -   **活跃线程**
        -   活动的 Java 线程总是被认为是活的对象，因此是 GC 根。这一点对于线程局部变量尤为重要。
    -   **静态变量**
        -   静态变量是由它们的类引用的。这一事实使它们成为事实上的GC根。类本身可以被垃圾回收，这将删除所有引用的静态变量。当我们使用应用服务器、OSGi 容器或一般的类加载器时，这一点特别重要。
    -   **JNI 引用**
        -   JNI 引用是本地代码作为 JNI 调用的一部分而创建的 Java 对象。这样创建的对象被特殊对待，因为 JVM 不知道它是否被本地代码引用。这种对象代表了 GC 根的一种非常特殊的形式。

### Stop-The-World(STW) pause in JVM

**不同的事件会导致 JVM 暂停所有的应用线程。这样的暂停称为：`Stop-The-World(STW) Pause`。**

#### SafePoint(安全点)

- **Oracle 官方解释：**A point during program execution at which all GC roots are known and all heap object contents are consistent. From a global point of view, all threads must block at a safepoint before the GC can run.
- **STW 暂停机制被称为 Safepoint(安全点)，安全点是程序执行中的一个点，在这个点上，程序的状态是已知的，可以检查。比如寄存器、内存等。JVM要想完全暂停和运行任务（如GC），所有线程必须来到一个安全点。**
  - 例如，要检索一个线程上的堆栈跟踪，我们必须来到一个安全点。这也意味着像 `jstack` 这样的工具要求程序的所有线程都能够到达一个**安全点**。

#### 触发 STW 暂停的最常见原因：

1. **GC Collection 操作**
2. **JIT 操作**
3. **偏向锁撤销**
4. **JVMTI(JVM Tool Interface / JVM 提供的 native 接口 )** 
5. **其他操作（死锁检查、stacktrace 转储等）**

- `-XX:+LogVMOutput -XX:LogFile=vm.log` 记录 JVM 日志
- **[垃圾收集代码 STW 演示](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/safepoints/FullGc.java)**

```java
import java.util.ArrayList;
import java.util.Collection;

public class FullGc {

    private static final Collection<Object> leak = new ArrayList<>();
    private static volatile Object sink;

    // Run with: -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails
    // Notice that all the stop the world pauses coincide with GC pauses

    public static void main(String[] args) {
        while(true) {
            try {
                leak.add(new byte[1024 * 1024]);

                sink = new byte[1024 * 1024];
            } catch(OutOfMemoryError e) {
                leak.clear();
            }
        }
    }
}

// 输出结果：
// Application time 在显示 0.0544056 秒的时间内完成了有用的工作
// Total time for which application threads were stopped 在应用程序完成有用工作后，所有线程暂停了 0.0174987 秒
// Stopping threads took 其中停止线程花费了 0.0000163 秒
Application time: 0.0544056 seconds
Total time for which application threads were stopped: 0.0174987 seconds, Stopping threads took: 0.0000163 seconds
Application time: 0.0067839 seconds
Total time for which application threads were stopped: 0.0246979 seconds, Stopping threads took: 0.0000130 seconds
```

-  **[偏向锁代码 STW演示](https://github.com/gvsmirnov/java-perv/blob/master/labs-8/src/main/java/ru/gvsmirnov/perv/labs/safepoints/BiasedLocks.java)**
  - **偏向锁核心思想：当线程 T1 获取锁后，再次申请持有锁时，无需再执行获取锁的操作。节省了申请锁的操作，提升了性能。如果其他线程再来申请锁，T1 会退出偏向锁模式。但是当大量线程不断请求锁，会导致线程竞争激烈，反而会降低性能。**

```java
import java.util.concurrent.locks.LockSupport;
import java.util.stream.Stream;

public class BiasedLocks {

    private static synchronized void contend() {
        LockSupport.parkNanos(100_000);
    }

    // Run with: -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails
    // Notice that there are a lot of stop the world pauses, but no actual garbage collections
    // This is because PrintGCApplicationStoppedTime actually shows all the STW pauses

    // To see what's happening here, you may use the following arguments:
    // -XX:+PrintSafepointStatistics  -XX:PrintSafepointStatisticsCount=1
    // It will reveal that all the safepoints are due to biased lock revocations.

    // Biased locks are on by default, but you can disable them by -XX:-UseBiasedLocking
    // It is quite possible that in the modern massively parallel world, they should be
    // turned back off by default

    public static void main(String[] args) throws InterruptedException {

        Thread.sleep(5_000); // Because of BiasedLockingStartupDelay

        Stream.generate(() -> new Thread(BiasedLocks::contend))
                .limit(10)
                .forEach(Thread::start);
    }

}

// 参数：
+PrintGCApplicationStoppedTime 打印 GC 暂停持续时间
+PrintGCDetails 打印 GC 详细信息
+PrintSafepointStatistics 打印安全点信息
+PrintSafepointStatisticsCount 打印安全点次数  
+UseBiasedLocking 开启偏向锁（JVM 默认是开启状态）
-UseBiasedLocking 关闭偏向锁  
  
// 使用以下参数返回的标识解释：
vmop(VM Operation)
  threads(Safepoint 的第一时间戳)
	  total	安全点内的线程总数
  	initially_running	安全点开始时，正在运行状态的线程数
	  wait_to_block	在 vmop 开始前需要等待其暂停的线程数
  time(到达 Safepoint 的时间戳)
  	spin	等待线程响应 Safepoint 号召的时间
  	block	暂停所有线程所有时间
  	sync	从开始进入 Safepoint 所耗时间 sync = spin + block
  	cleanup 清理所用时间
  	vmop	真正执行 VM Operation 所耗时间
  page_trap_count 页面陷阱技术
Metaspace
	used 显示用于加载类的空间量
	capacity 当前已分配的块中 Metaspace 可用的空间量
	committed 块的可用空间量
  reserved	为 Metaspace 保留的空间量
  
// 参数设置  
1. -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails
2. -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails -XX:+PrintSafepointStatistics  -XX:PrintSafepointStatisticsCount=1  
// 1、2 的设置，可以查看 STW 次数一致，3关闭了偏向锁，STW 次数明显减少  
3. -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails -XX:+PrintSafepointStatistics  -XX:PrintSafepointStatisticsCount=1 -XX:-UseBiasedLocking 
```

### 垃圾收集算法

#### Mark & Sweep(标记 & 扫描)

>   **删除未引用对象的基本策略是：识别存活对象并删除所有剩余对象。这分为两个阶段：Mark 和 Sweep。**

**任何垃圾收集算法都需要具备三个基本步骤：**

1.  **Mark(标记阶段)**
    -   查找所有可能在未来被使用的对象
2.  **Sweep / Copy (扫描 / 复制阶段)**
    -   删除所有未使用的对象
3.  **Compact(紧凑阶段)**
    -   一般为压缩操作，防止产生内存碎片。

### 垃圾收集器
