# 识别 OutOfMemory

![Memory-Leaks](https://gitee.com/bonismo/notebook-img/raw/master/img/jvm/memory-leaks.jpg)

经验不足的程序员经常认为 **Java** 的 `GC` 使他们完全不必担心内存管理。这是一个普遍的误解：尽管垃圾收集器会尽力而为，但即使是最好的程序员，也有可能陷入严重的内存泄漏。

## 1. 内存溢出与内存泄漏

**Out Of Memory(内存溢出)：**

-   **应用程序向 JVM 申请内存时，无法申请足够的内存空间供其使用而导致的错误。**内存溢出通常发生在 `Old Gen` 或 `Metaspace`。
-   异常表现：`OutOfMemoryError`

**Memory Leaks(内存泄露)：**

-   **应用程序中一些对象不再被使用，但是垃圾收集器始终无法清除，始终占用内存空间，即已分配对象不使用但是仍然占用内存空间。这种现象如果持续发生，则会耗尽内存。**
-   异常表现：`OutOfMemoryError`

所以当应用程序出现 **OutOfMemoryError** 异常，则需要分析出具体是哪一种情况，才可以提出合适的解决方案。

## 2. OutOfMemoryError 各种情况及原因

识别 `OutOfMemoryError` 则首先需要了解发生该异常的所有情况。以下摘自：[Oracale Understand the OutOfMemoryError Exception](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)

###  2.1 Java heap space

>   **异常原因：`Heap Memory` 无法分配对象。**

如果只是因为 `Heap Memory` 空间不足，可以使用 `-Xmx` 和 `-Xms` 参数设置最大和初始值。

此错误还有可能是**内存泄漏**，该问题接下来详细解释。

**代码演示**

```java
// -Xmx10m -Xms10m -XX:+UseParallelGC
public class OutOfMemoryExample {
    
    public static void main(String[] args) {
        OutOfMemoryExample ome = new OutOfMemoryExample();
        ome.generateMyIntArray(1, 51);
    }

    public void generateMyIntArray(int start, int end) {
        int multiplier = 100;
        for (int i = start; i < end; i++) {
            System.out.println("Round " + i + " Free Memory: " + Runtime.getRuntime().freeMemory());
            int[] multiList = new int[multiplier];
            for (int j = i; j >= 1; j--) {
                multiList[j] = i;
            }
            multiplier = multiplier * 10;
        }
    }
}
```

### 2.2  GC Overhead limit exceeded

>   **超出了GC开销限制。**

GC一直在运行，并且程序的运行非常缓慢。进行垃圾回收之后，如果 Java 进程花费了其大约 **98％** 的时间用于垃圾回收，并且正在恢复的内存少于 **2％**，并且到目前为止已经执行了最后 **5个（编译时间常数）**连续垃圾集合，然后抛出异常，因为 `Heap Memory` 剩余空间几乎无法容纳新的数据，而新分配的可用空间却很少。

**简言之，GC 是一种持续运行的状态，因为每次 GC 后的内存空间马上被新申请的空间所使用，于是 GC 不断循环回收内存空间，又不断的被使用。**

Oracle 建议使用 `-XX:-UseGCOverheadLimit` 关闭该警告异常，不太建议这么做。因为一旦关闭，到一定程度仍然抛出异常，只不过具体原因变成了 `java.lang.OutOfMemoryError: Java heap space`，这会增加排查 `OOM ` 的难度。

**代码演示**

```java
// -Xmx10m -Xms10m -XX:+UseParallelGC		
public class GCOverheadExample {

    public static void main(String[] args) throws Exception {
        Map<Object, Object> map = System.getProperties();
        Random r = new Random();
        while (true) {
            map.put(r.nextInt(), "GC Overhead Limit Test");
        }
    }
}
```

### 2.3 Requested array size exceeds VM limit

### 2.4 Metaspace

### 2.5 request size bytes for reason. Out of swap space?

### 2.6 Compressed class space

### 2.7 reason stack_trace_with_native_method