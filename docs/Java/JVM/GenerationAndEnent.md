# Heap Memory 的分代模型与 GC 事件

`Heap Memory` 产生分代模型，是基于 `GC` 分代治理的核心理念建立起来的。具体可参考：[JVM 内存模型](http://notebook.bonismo.ink/#/Java/JVM/JVM)

## Heap Memory 的分代模型

### Weak Generational Hypothesis(弱代际假设)

![Lifetime](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Object-Lifetime.png)

自 1984 年来，通过对 JVM 内部对象**生命周期**进行了很长时间的观察和研究，大多数对象在刚分配内存后多久就死掉了（不再被引用），而一旦这些对象存活到一定的年龄后**（每次 GC 增加一次年龄）**，又会存活很长一段时间。**简言之：大多数分配的对象都死于年轻时。**

**文献**

[Generation Scavenging 论文](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.122.4295&rep=rep1&type=pdf)

[Oracle Generations 文档](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/generations.html)

### Heap Memory 的分代

![HeapMemory](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/HeapMemory.png)

`Heap Memory` 基于[垃圾删除策略](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_4-%e6%a0%87%e8%ae%b0amp%e6%89%ab%e6%8f%8famp%e5%8e%8b%e5%ae%9eamp%e5%a4%8d%e5%88%b6%e7%ad%96%e7%95%a5)，可以根据对象存活时间的长短，而使用不同的策略，进而根据 `弱代际假设` 理论进行分代。

**垃圾收集器**的最终目标是根据对象存活时间长短在垃圾收集的过程中尽可能的增加 `Application throught(应用吞吐量)` 同时，并减少因为 [STW](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_22-stop-the-worldstw-pause-in-jvm) 产生的 `Application pause(应用暂停)` ，`Heap Memory` 被划分为 `Young Gen` 和 `Old Gen`。

`Young Gen` 因为[垃圾收集策略](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_4-%e6%a0%87%e8%ae%b0amp%e6%89%ab%e6%8f%8famp%e5%8e%8b%e5%ae%9eamp%e5%a4%8d%e5%88%b6%e7%ad%96%e7%95%a5)采用了 `Mark-Copy` 算法，进而需要再次划分 `Eden` 、`Survivor0`、`Survivor1` 三个区域。**该堆区 GC 事件称为 `Minor GC`。**

`Old Gen` 因为[垃圾收集策略](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_4-%e6%a0%87%e8%ae%b0amp%e6%89%ab%e6%8f%8famp%e5%8e%8b%e5%ae%9eamp%e5%a4%8d%e5%88%b6%e7%ad%96%e7%95%a5)采用了 `Mark-Sweep-Compact` 算法，所以无需像 `Young Gen` 一样再次划分。**该堆区 GC 事件称为 `Major GC`。**

**整个 `Heap Memory(Young Gen + Old Gen + Metasapce)` 区的 GC 事件称为 Full GC。**


## Heap Memory 内的 GC 事件

### Old Gen 的分配担保机制

**JVM 在每次 `Minor GC` 之前都会有两个判断条件：**

1. 判断 `Old Gen` 的最大可用连续空间是否大于 `Young Gen` 的存活对象空间，如果大于则进行 `Minor GC`，否则进行 `Full GC`。
2. 判断 `Old Gen` 的最大可用连续内存空间是否大于**每次移至老年代的对象的平均大小**，如果大于，直接尝试 `Minor GC` 否则进行 `Full GC` 之后再进行 `Minor GC`，需要注意的是如果 `Full GC` 之后的空间仍然无法存放对象，则会导致 `OutOfMemoryError`。

**该机制主要防止在 `Minor GC` 之后，移动存活对象到 `Old Gen` 因为内存空间不够而导致 `OutOfMemoryError`**

`-XX:HandlePromotionFailure`  JVM 参数在 JDK 1.6u24 之后**失效**，在 JDK 7 之后默认设置为 **true**。

### Minor & Major & Full GC 事件机制

**Google 到的一张图，GC 事件过程描述了 Minor GC 和 Full GC 的流程图（缺少了 Major GC，因为官方定义也不明确，可以暂时按照下图理解，遇到明确的资料再更新），图中 HandlePromotionFailure = false 的流程请忽略（如有侵权，请联系删除）。**

![GC-Eevnt](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/GC-Event.png)

**下边文字描述的是 Minor & Major & Ful GC。与上图无关，请单独理解。**

#### Minor GC

1. JVM 每次分配对象（进入 `Old Gen` 有 **4** 种情况，具体请参考 [JVM 内存模型 Young Gen](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_1211-young-gen%e5%b9%b4%e8%bd%bb%e4%bb%a3%ef%bc%8c%e5%8d%a0%e7%94%a8-heap-%e7%9a%84-13)），会首先将对象分配到 `Eden` 区。当 JVM 无法为新对象分配空间时，触发 `Minor GC`，存活下来的对象，将被复制到其中一个空闲的 `Survivor` 区，在此次触发过程中，会**存活对象增加一次年龄**。`Minor GC` 进行垃圾回收时，总会检查 `Survivor` 区，将区内存活对象移至林那个一个 `Survivor` 区，保证总是有一个 `Survivor` 区域是空闲的。**`Minor GC` 会造成 JVM 的 `STW`，通常因为时间很短，可以忽略不计。**
   - 直接进入 `Old Gen` 的**4** 种情况
     1. 默认 GC 年龄超过 **15** 次进入 `Old Gen`
     2. 动态对象年龄，`Eden` 区存活对象总大小超过 `Survivor` **50%** 的空间，则年龄最大的对象进入 `Old Gen`
     3. 大对象（比如：大字符串或数组）直接进入 `Old Gen`
     4. `Survivor` 区域无法储存 `Eden` 区存活对象直接进入 `Old Gen`

#### Major GC

1. 经过多次 `Minor GC` 周期（默认 **15** 次），`Young Gen` 区域存活下来的对象将被移至 `Old Gen` 区域。当 `Old Gen` 区域内没有多余空间时，就会触发 `Major GC`，耗时比 `Minor GC`  多。**`Major GC` 造成的 JVM 的 `STW` 时间较长，在 JVM 优化中需要注意。**

#### Full GC

1. 当 `Minor GC` 之后的对象，因连续内存空间太大无法移至 `Survivor` 区，触发 `Major GC`，在 `Major GC` 之后 ，`Old Gen` 剩余连续空间**小于**需要存储的存活对象或者 `Old Gen` **小于**存活对象的平均大小（分配担保机制），则会触发 `Full GC`，如果触发 `Full GC` 之后仍然无法存储，则导致 `OutOfMemoryError`。
2. 在 JDK 8 中，当 `Metaspace` 的内存空间超过默认值**（20MB）**时，也会触发 `Full GC` 进行回收。在 `Full GC` 后，会增加 `Metaspace` 的大小，以延迟因为存储空间而导致的下次 `Full GC` 时间。所以设置一个合理的 `MaxMetaspaceSize` 也很重要。
   - `Metaspace` 回收的对象**必须同时满足 3 个条件**
     1. 该类的所有实例已经被回收。
     2. 加载该类的 ClassLoader 已经被卸载。
     3. 该类对应的 java.lang.Class 对象没有被引用，因为 `Heap Memory` 内的对象持有对 `Metaspace` 对象的特殊引用。

