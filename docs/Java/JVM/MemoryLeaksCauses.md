# 追踪 OutOfMemory 

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

## 2. OutOfMemoryError 常见情况及原因

[Oracle 故障排除指南](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html)

识别 `OutOfMemoryError` 则首先需要了解发生该异常的所有情况。以下摘自：[Oracle Understand the OutOfMemoryError Exception](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)

###  2.1 Java heap space

>   **异常原因：`Heap Memory` 无法分配对象，则会引发`java.lang.OutOfMemoryError: Java heap space`。**

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

>   **超出了GC开销限制，则会引发 `java.lang.OutOfMemoryError: GC Overhead limit exceeded`。**

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

> **请求的数组大小超出 VM 限制，表示该应用程序（或该应用程序使用的API）试图分配一个大于当前系统允许的 Heap 大小的数组，则会引发 ` java.lang.OutOfMemoryError: Requested array size exceeds VM limit`。**

例如，如果应用程序尝试分配 **512 MB** 的数组，但最大堆大小为 **256 MB**，`OutOfMemoryError`则将抛出该数组，原因是请求的数组大小超出了 JVM 限制。

**代码演示**

```java
public class ArraySizeLimitExample {

    public static void main(String[] args) {
        int[] intArray = new int[Integer.MAX_VALUE - 1];
    }
}
```

### 2.4 Metaspace

> **Java 的 `class metaspace(类元数据)`（Java类的虚拟机内部表示形式）分配在本机内存（Metaspace）中。如果类元数据的 `Metaspace` 已用尽，则会引发 ` java.lang.OutOfMemoryError: Metaspace`。**

可用于类元数据的元空间的数量受 `-XX:MaxMetaSpaceSize=***m` 在 JVM 上指定的参数的限制。当类元数据所需的本机内存量超过指定的 `MaxMetaSpaceSize` 导致异常。

**代码演示1**

```java
// add maven dependencies
        <dependency>
            <groupId>cglib</groupId>
            <artifactId>cglib</artifactId>
            <version>3.2.6</version>
        </dependency>

// -XX:MaxMetaspaceSize=100m -XX:MetaspaceSize=2M -XX:MaxMetaspaceFreeRatio=1 -XX:MaxMetaspaceExpansion=1K -XX:MinMetaspaceFreeRatio=1 -XX:InitialBootClassLoaderMetaspaceSize=2M
public class MetaspaceExample {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(MetaspaceExample.class);
            enhancer.setCallbackTypes(new Class[]{Dispatcher.class, MethodInterceptor.class});
            enhancer.setCallbackFilter(new CallbackFilter() {
                @Override
                public int accept(Method method) {
                    return 1;
                }

                @Override
                public boolean equals(Object obj) {
                    return super.equals(obj);
                }
            });
            Class clazz = enhancer.createClass();
        }
    }
}  
```

**代码演示2**

JDK8 无法运行，需要在 JDK11 运行。

> JDK8 日志
>
> Exception in thread "main" java.lang.UnsupportedClassVersionError: oom/MetaspaceExample0 has been compiled by a more recent version of the Java Runtime (class file version 55.0), this version of the Java Runtime only recognizes class file versions up to 52.0
>
> **Java 主版本号映射**
>
> - 45 = Java 1.1
> - 46 = Java 1.2
> - 47 = Java 1.3
> - 48 = Java 1.4
> - 49 = Java 5
> - 50 = Java 6
> - 51 = Java 7
> - 52 = Java 8
> - 53 = Java 9
> - 54 = Java 10
> - 55 = Java 11
> - 56 = Java 12
> - 57 = Java 13

```java
// -XX:MaxMetaspaceSize=100m -XX:MetaspaceSize=2M -XX:MaxMetaspaceFreeRatio=1 -XX:MaxMetaspaceExpansion=1K -XX:MinMetaspaceFreeRatio=1 -XX:InitialBootClassLoaderMetaspaceSize=2M
public class MetaspaceExample {

    public static void main(String[] args) throws Exception {
        String clazzBase64 = "yv66vgAAADcADAEAEm15cGFja2FnZS9NeWNsYXNzMAcAAQEAEGphdmEvbGFuZy9PYmplY3QHAAMBAApTb3VyY2VGaWxlAQANTXljbGFzczAuamF2YQEABjxpbml0PgEAAygpVgwABwAICgAEAAkBAARDb2RlACEAAgAEAAAAAAABAAEABwAIAAEACwAAABEAAQABAAAABSq3AAqxAAAAAAABAAUAAAACAAY=";

        byte[] compiledClazz = Base64.getDecoder().decode(clazzBase64);
        int classNameLength = Integer.valueOf(compiledClazz[12]);

        MyClassLoader myClassLoader = new MyClassLoader(Thread.currentThread().getContextClassLoader());

        for (int i = 0; ; i++) {
            byte[] bytes = String.valueOf(i).getBytes();
            byte[] bytecode = new byte[compiledClazz.length + bytes.length - 1];
            System.arraycopy(compiledClazz, 0, bytecode, 0, 30);
            bytecode[12] = (byte) (classNameLength + bytes.length - 1 & 0xFF);

            System.arraycopy(bytes, 0, bytecode, 30, bytes.length);
            System.arraycopy(compiledClazz, 31, bytecode, 30 + bytes.length, compiledClazz.length - 31);

            String classname = "oom.MetaspaceExample" + i;
            Class c = myClassLoader.getClass(classname, bytecode);
        }
    }

    public static class MyClassLoader extends ClassLoader {
        public MyClassLoader(ClassLoader parent) {
            super(parent);
        }

        public Class<?> getClass(String name, byte[] code) {
            return defineClass(name, code, 0, code.length);
        }
    }
}
```

### 2.5 unable to create new native thread

> **JVM 需要创建一个新线程，但是它失败了，最常见的原因是已经达到了进程限制。 则会引发 `java.lang.OutOfMemoryError: unable to create new native thread`。**

**代码演示**

```java
public class NativeThreadExample {
    
    public static void main(String[] args) {

        while (true) {
            NativeThread nativeThread = new NativeThread();
            nativeThread.start();
        }
    }
}

class NativeThread extends Thread {

    public void run() {
        try {
            Thread.sleep(10000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 2.6 Direct buffer memory

> **申请的堆外内存空间超出限制，则会引发 `java.lang.OutOfMemoryError: Direct buffer memory`。**

可以通过 JVM 参数 `-XX:MaxDirectMemorySize=1G` 设置堆外内存大小，默认堆外内存是 **64 M**。

**代码演示**

```java
// -XX:MaxDirectMemorySize=200M
public class DirectMemoryExample {

    public static void main(String[] args) {
        List<ByteBuffer> list = new ArrayList<>();
        while (true) {
            ByteBuffer bb = ByteBuffer.allocateDirect(1024 * 1024);
            list.add(bb);
        }
    }
}
```

### 2.7 request size bytes for reason. Out of swap space?

> 请求大小字节。交换空间不足？似乎是一个例外。但是，当来 `Native Heap(本机堆)` 的分配失败并且 `Native Heap(本机堆)` 接近耗尽时，Java HotSpot VM 代码会报告此明显的异常。该消息指示失败的请求的大小（以字节为单位）以及内存请求的原因。通常，原因是报告分配失败的源模块的名称，尽管有时是实际原因。`OutOfMemoryError`

抛出该错误消息时，虚拟机会调用致命错误处理机制（即生成一个致命错误日志文件，其中包含了崩溃时线程、进程和系统的有用信息）。在本机堆耗尽的情况下，日志中的堆内存和内存映射信息可能是有用的。可能需要使用操作系统上的故障排除实用程序来进一步诊断问题，具体参考 [Native Operating System Tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr020.html#BABBHHIE)

### 2.8 Compressed class space

> **在 64 位平台上，指向类元数据的指针可以用 32 位偏移来表示（使用UseCompressedOops）。这是由 JVM 参数 `-XX:+UseCompressedClassPointers`（默认开启）控制的。如果使用该参数，则类元数据的可用空间量固定为 `-XX:CompressedClassSpaceSize=1G` (默认值 1G) 大小。如果超过 CompressedClassSpaceSize 设置的大小，则会引发 ` java.lang.OutOfMemoryError: Compressed class space`。**

可以使用 `-XX:-UseCompressedClassPointer` 关闭。或者设置更大的压缩类指针空间大小 `-XX:CompressedClassSpaceSize=10G`，**该值最小设置为 1G，最大设置为 3072G。**

### 2.9 reason stack_trace_with_native_method

> 如果打印了一个 `Stack` 跟踪，其中最上面的一帧是本地方法，那么这表明 `Native Method(C 代码)` 遇到了分配失败。分配失败是在 `Java Native Interface (JNI)` 或 `Native Method` 中检测到的，而不是在 JVM 的相关代码中。

## 3. Memory Leaks 常见原因

