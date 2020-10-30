# Heap Memory 的分代模型与 GC 事件

`Heap Memory` 产生分代模型，是基于 `GC` 分代治理的核心理念建立起来的。具体可参考：[JVM 内存模型](http://notebook.bonismo.ink/#/Java/JVM/JVM)

## Heap Memory 的分代模型

### Weak Generational Hypothesis(弱代际假设)

![Lifetime](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/Object-Lifetime.png)

自 1984 年来，通过对 JVM 内部对象**生命周期**进行了很长时间的观察和研究，大多数对象在刚分配内存后多久就死掉了（不再被引用），而一旦这些对象存活到一定的年龄后（每次 GC 增加一次年龄），又会存活很长一段时间。**简言之：大多数分配的对象都死于年轻时。**

**文献**

[Generation Scavenging 论文](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.122.4295&rep=rep1&type=pdf)

[Oracle Generations 文档](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/generations.html)

### Heap Memory 的分代

![HeapMemory](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/HeapMemory.png)

`Heap Memory` 基于[垃圾删除策略](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_4-%e6%a0%87%e8%ae%b0amp%e6%89%ab%e6%8f%8famp%e5%8e%8b%e5%ae%9eamp%e5%a4%8d%e5%88%b6%e7%ad%96%e7%95%a5)，可以根据对象存活时间的长短，而使用不同的策略，进而根据 `弱代际假设` 理论进行分代。

**垃圾收集器**的最终目标是根据对象存活时间长短在垃圾收集的过程中尽可能的增加 `Application throught(应用吞吐量)` 同时，并减少因为 [STW](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_22-stop-the-worldstw-pause-in-jvm) 产生的 `Application pause(应用暂停)` ，`Heap Memory` 被划分为 `Young Gen` 和 `Old Gen`。

`Young Gen` 因为[垃圾收集策略](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_4-%e6%a0%87%e8%ae%b0amp%e6%89%ab%e6%8f%8famp%e5%8e%8b%e5%ae%9eamp%e5%a4%8d%e5%88%b6%e7%ad%96%e7%95%a5)采用了 `Mark-Copy` 算法，进而需要再次划分 `Eden` 、`Survivor0`、`Survivor1` 三个区域。**该堆区 GC 事件称为 `Minor GC`。**

`Old Gen` 因为[垃圾收集策略](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_4-%e6%a0%87%e8%ae%b0amp%e6%89%ab%e6%8f%8famp%e5%8e%8b%e5%ae%9eamp%e5%a4%8d%e5%88%b6%e7%ad%96%e7%95%a5)采用了 `Mark-Sweep-Compact` 算法，所以无需像 `Young Gen` 一样再次划分。**该堆区 GC 事件称为 `Major GC`。**

**整个 `Heap Memory(Young Gen + Old Gen)` 区的 GC 事件称为 Full GC。**


## Heap Memory 内的 GC 事件

### Minor GC

### Major GC

### Full GC
