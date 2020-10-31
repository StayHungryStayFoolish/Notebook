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

### GC 的分配担保机制

**JVM 在每次 `Minor GC` 之前都会有两个判断条件：**

1. 判断 `Old Gen` 的最大可用连续空间是否大于 `Young Gen` 的存活对象空间，如果大于则进行 `Minor GC`，否则进行 `Full GC`。
2. 判断 `Old Gen` 的最大可用连续内存空间是否大于**每次晋升到老年代的对象的平均大小**，如果大于，尝试 `Minor GC`否则进行 `Full GC` 。

**该机制主要防止在 `Minor GC` 之后，移动存活对象到 `Old Gen` 因为内存空间不够而导致 `OutOfMemoryError`**

### Minor & Major & Full GC 事件机制

1. JVM 每次分配对象（此处有 **4** 种情况会直接进入 `Old Gen`，具体请参考 [JVM 内存模型 Young Gen](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_1211-young-gen%e5%b9%b4%e8%bd%bb%e4%bb%a3%ef%bc%8c%e5%8d%a0%e7%94%a8-heap-%e7%9a%84-13)），会首先将对象分配到 `Eden` 区。当 JVM 无法为新对象分配空间时，触发 `Minor GC`，存活下来的对象，将被复制到其中一个空闲的 `Survivor` 区，在此次触发过程中，会**存活对象增加一次年龄**。`Minor GC` 进行垃圾回收时，总会检查 `Survivor` 区，将区内存活对象移至林那个一个 `Survivor` 区，保证总是有一个 `Survivor` 区域是空闲的。**`Minor GC` 会造成 JVM 的 `STW`，通常因为时间很短，可以忽略不计。**
2. 经过多次 `Minor GC` 周期（默认 **15** 次），`Young Gen` 区域存活下来的对象将被移至 `Old Gen` 区域。当 `Old Gen` 区域内没有多余空间时，就会触发 `Major GC`，耗时大于是 `Minor GC`  的 **10** 倍左右。**`Major GC` 造成的 JVM 的 `STW` 时间较长，在 JVM 优化中需要注意。**
3. 