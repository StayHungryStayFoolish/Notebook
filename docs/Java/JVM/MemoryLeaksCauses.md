# 追踪 OutOfMemory 

![Memory-Leaks](https://raw.githubusercontent.com/StayHungryStayFoolish/notebook-img/master/img/jvm/memory-leaks.jpg?raw=true)

经验不足的程序员经常认为 **Java** 的 `GC` 使他们完全不必担心内存管理。这是一个普遍的误解：尽管垃圾收集器会尽力而为，但即使是最好的程序员，也有可能陷入严重的内存泄漏。

## 1. 内存溢出与内存泄漏

**Memory Overfolw(内存溢出)：**

-   **应用程序向 JVM 申请内存时，无法申请足够的内存空间供其使用而导致的错误。**内存溢出通常发生在 `Old Gen` 或 `Metaspace`。
-   异常表现：`OutOfMemoryError`

**Memory Leak(内存泄露)：**

-   **应用程序中一些对象不再被使用，但是垃圾收集器始终无法清除，始终占用内存空间，即已分配对象不使用但是仍然占用内存空间。这种现象如果持续发生，则会耗尽内存。**
-   异常表现：`OutOfMemoryError`

所以当应用程序出现 **OutOfMemoryError** 异常，则需要分析出具体是哪一种情况，才可以提出合适的解决方案。

## 2. OutOfMemoryError 常见情况及原因

[Oracle 故障排除指南](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html)

识别 `OutOfMemoryError` 则首先需要了解发生该异常的所有情况。以下摘自：[Oracle Understand the OutOfMemoryError Exception](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)

###  2.1 Java heap space

>   **异常原因：`Heap Memory` 无法分配对象，则会引发`java.lang.OutOfMemoryError: Java heap space`。**

如果只是因为 `Heap Memory` 空间不足，可以使用 `-Xmx` 和 `-Xms` 参数设置最大和初始值。

此错误还有可能是**内存泄漏**，该问题将在[Memory Leaks 常见原因](http://notebook.bonismo.ink/#/Java/JVM/MemoryLeaksCauses?id=_3-memory-leaks-%e5%b8%b8%e8%a7%81%e5%8e%9f%e5%9b%a0)解释。

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

### 3.1 可变的静态字段和集合

`static` 修饰的**可变字段和集合**在 JVM 内存结构中实际上是 `GC Root`，具体可以查看 [Java Garbage Collection Roots](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_21-java-garbage-collection-rootsgc-roots-java-%e5%9e%83%e5%9c%be%e6%94%b6%e9%9b%86%e6%a0%b9)。

意味着永远不会被 GC 回收。`static` 修饰的可变字段或集合通常用来保持缓存或跨线程共享状态。`static` 的可变字段和集合需要**在编码中显示清理（当不再使用时，显示设置为 null 等处理）**。在应用程序中，如果没有考虑这种可能性，GC 不会进行回收，可能会造成内存泄漏。

**在编程中建议只有使用常量（final static），避免使用可变的字段或集合。**

**代码演示**

```java
public class StaticLeak {
    public static List<Double> list = new ArrayList<>();
 
    public void populateList() {
        for (int i = 0; i < 10000000; i++) {
            list.add(Math.random());
        }
    }
 
    public static void main(String[] args) {
        new StaticLeak().populateList();
    }
}
```

### 3.2 未关闭的流

在进行 IO 操作时，如果不能正确的对流进行关闭，则会一直保持对象的引用，GC 不能进行对其回收，可能会造成内存泄漏。

**在编程中对 IO 操作一定要在 `finally` 中进行关闭。**

**代码演示**

```java
public class URLeak {
 
    public static void main(String[] args) throws IOException {
        URL url = new URL("https://raw.githubusercontent.com/zemirco/sf-city-lots-json/master/citylots.json");
        ReadableByteChannel rbc = Channels.newChannel(url.openStream());
        FileOutputStream outputStream = new FileOutputStream("./citylots.json");
        outputStream.getChannel().transferFrom(rbc, 0, Long.MAX_VALUE);
    }
}
```

### 3.3 未关闭的连接

在连接第三方服务时，如果连接未关闭，可能会造成内存泄漏。

**例如原生方式连接数据库一定要在 `finally` 中进行关闭。建议使用 `ORM` 框架，因为会帮我们进行连接资源的关闭。**

**代码演示**

```java
public class DatabaseLeak {

    public static final String URL = "jdbc:mysql://localhost:3306/mydb";
    public static final String USER = "root";
    public static final String PASSWORD = "admin";

    public static void main(String[] args) throws Exception {
        Class.forName("com.mysql.jdbc.Driver");
        Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
        PreparedStatement stmt = conn.prepareStatement("SELECT ... ...");
    }
}
```

### 3.4 equals() 和 hashCode() 重写不当

一个非常常见的疏忽是在定义一个类时没有为 `equals()` 和 `hashCode()` 方法进行适当的重写。

`HashSet` 和 `HashMap` 在许多操作中都使用这些方法，如果未正确重写它们，则它们可能成为潜在内存泄漏问题的根源。

**在定义类时，始终重写 `equals()` 和 `hashCode()` 方法。**

**代码演示**

```java
public class Person {
    public String name;

    public Person(String name) {
        this.name = name;
    }

//    @Override
//    public boolean equals(Object o) {
//        if (o == this) return true;
//        if (!(o instanceof Person)) {
//            return false;
//        }
//        Person person = (Person) o;
//        return person.name.equals(name);
//    }
//
//    @Override
//    public int hashCode() {
//        int result = 17;
//        result = 31 * result + name.hashCode();
//        return result;
//    }
  
}	

public class OverrideLeak {
    public static void main(String[] args) {
        Map<Person, Integer> map = new HashMap<>();
        for (int i = 0; i < 100000; i++) {
            // 使用 person 作为 key，如果不使用 equals() 和 hashCode()，map 的 size 最终是100000
            map.put(new Person("boni"), i);
        }
	      System.out.println(map.size());
    }
}
```

### 3.5 引用外部类的内部类

当引用一个`内部类对象（非静态内部类）`时，该类的初始化会默认需要初始化一个当前内部类的`外部类的实例`。内部类对象会持有一个`外部类`的**隐式引用**。如果`内部类`比`外部类`生命周期长的情况下，也不会被 GC 进行回收。

**当定义内部类时，可以使用 `staic` 使内部类变为静态内部类或者使用匿名内部类，也可以使用 `WeakReference` 使内部类作为弱引用。**

```java
// 可以使用 javac 编译 OuterClass 会发现有两个类文件，分别是 OuterClass.class 和 OuterClass$InnerClass.class
public class OuterClass {
    private int[] data;

    public OuterClass(int size) {
        data = new int[size];
    }

    class InnerClass {
    }

    InnerClass getInnerClassInstance() {
        return new InnerClass();
    }
}

public class InnerClassLeak {

    public static void main(String[] args) {
        List<OuterClass.InnerClass> list = new ArrayList<>();
        int counter = 0;
        while (true) {
            list.add(new OuterClass(1).getInnerClassInstance());
            System.out.println(counter++);
        }
    }
}
```

**Android 使用内部类也会出现内存泄漏问题**

```java
class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        new MyAsyncTask().execute();
    }

    private class MyAsyncTask extends AsyncTask {
        @Override
        protected Object doInBackground(Object[] params) {
            return doSomeStuff();
        }
        private Object doSomeStuff() {
            //do something to get result
            return new MyObject();
        }
    }
}
```

### 3.6 重写 finalize() 方法

使用 `@Override` 重写 `Object` 对象内的 `finalize()` 方法，因为对象最终将对 JVM 的 `finalizer(终结器)` 保持可见，不会被 GC 回收，可能会造成内存泄漏。具体可以参考 [JIT 的逃逸算法-全局逃逸情况](http://notebook.bonismo.ink/#/Java/JVM/EscapeAnalysis?id=_4-%e5%9f%ba%e4%ba%8e-ea-%e6%a0%87%e8%af%86%e7%9a%84%e5%af%b9%e8%b1%a1%e7%8a%b6%e6%80%81)

### 3.7 线程局部变量

线程局部变量基本上是线程类中的一个成员字段。

在多线程应用程序中，每个线程实例都有自己的类变量实例。因此只要线程本身还处于活跃状态，GC 就不会删除该线程的局部变量。由于线程通常是池化的，因此几乎永远活着，这个对象可能永远不会 GC 删除。

一个活跃的线程被认为是 `GC Root`，所以线程局部变量与静态变量非常相似。需要在编码中进行**显式清理**。

这类内存泄漏可以通过`堆转储`来发现。查看堆转储中的 `ThreadLocalMap`，然后按照引用进行查找。也可以查看线程名称，确定内存泄漏的具体原因。

**如果多线程中使用 `ThreadLocal` 一定要在 `finally` 中使用 `remove()`，使用 `Thread.set(null)` 实际上并不会清除值，而是将 `key-vaule` 分别设置为 `threadId:null`。**

**代码演示**

```java
try {
    threadLocal.set(System.nanoTime());
}
finally {
  	// 必须使用 threadLocal.remove()，不能使用 threadLocal.set(null);
    threadLocal.set(null);
}
```

### 3.8 JNI 内存泄漏

`Java Native Method Interface(JNI)` 产生的内存泄漏很难发现。

`JNI` 与 `Native Method Library(本地方法库，通常由 C/C++ 编写)` 交互，通常由 Java 代码进行调用和创建Java对象。在 `JNI` 中创建的每个 Java 对象都以本地引用开始其生命，这意味着该对象将被引用，直到本机方法返回为止。可以理解为本机方法引用了 Java 对象。

**发现 `JNI` 内存泄漏的唯一方法是使用显式标记本机引用的堆转储工具。**

### 3.9 循环引用和复杂的双向引用（已过时）

在 `Refernence Counting Algorithm` 算法中存在内存泄漏偶问题，当使用了`Reachability Ananlysis Algorithm` 算法时已经解决了该问题。具体可以参考 [JVM 内存模型-垃圾分析算法](http://notebook.bonismo.ink/#/Java/JVM/JVM?id=_311-reference-counting-algorithm%e5%bc%95%e7%94%a8%e8%ae%a1%e6%95%b0%e7%ae%97%e6%b3%95)

### 3.10 String.intern() 方法（已过时）

在 JDK8 以后，因为使用了 `Metaspace` 替换了 `PermGen`，所以使用 `intern()` 方法生成的 `interned string` 存储在 `Metaspace`，不会出现内存泄漏。[JDK interned string bug](https://bugs.openjdk.java.net/browse/JDK-8180048)

## 4. 检测内存泄漏

**检测内存泄漏需要使用不同的工具和技术相结合。可以尝试使用以下步骤解决问题。**

1. 为应用程序配置详细的 GC 日志，使用 JVM 打印 GC 日志参数 `-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=100 -XX:GCLogFileSize=50m -Xloggc:/path/gc.log`，上述命令只是在指定路径滚动输出日志，并不包括具体 GC 打印参数，详细参数请参考 [JVM 调优参数表](http://notebook.bonismo.ink/#/Java/JVM/PerformanceTuningGuide?id=_41-jvm-%e8%b0%83%e4%bc%98%e5%8f%82%e6%95%b0%e8%a1%a8)。
2. 生成 `堆转储文件（以 .hprof 后缀）`。
3. 使用工具分析堆转储文件，并根据可以数据最终找出问题所在（一般大多数为代码问题，具体参考本文：**Memory Leaks 常见原因**）。

### 4.1 在线查看堆直方图

#### 4.1.1 jcmd 查看

`jcmd <pid> GC.class_histopram`

**jcmd 示例：**

```shell
λ ~/ jcmd 24499 GC.class_histogram
24499:

 num     #instances         #bytes  class name
----------------------------------------------
   1:          4112         367752  [C
   2:           499         136648  [B
   3:          4097          98328  java.lang.String
   4:           723          82776  java.lang.Class
   5:           709          45064  [Ljava.lang.Object;
   6:           107          40232  java.lang.Thread
   7:           696          27840  java.util.LinkedHashMap$Entry
   8:           574          18368  java.util.HashMap$Node
   9:           225          14400  java.net.URL
  10:           293          13320  [Ljava.lang.String;
  11:           167          12024  java.lang.reflect.Field
  12:            32          11264  [Ljava.util.HashMap$Node;
  13:           102           8160  [Ljava.lang.ThreadLocal$ThreadLocalMap$Entry;
  
# Class name 说明  
[C -> char[]
[B -> byte[]
[S -> short[]
[I -> int[]
[[I -> int[][]
[Ljava.lang.Object -> Object array
```

#### 4.1.2 jmap 查看

`jmap -histo <pid>` 查看所有对象

`jmap -histo:live <pid>` 先进行 `Full GC` 然后统计存活对象

**jmap 示例：**

```shell
λ ~/ jmap -histo  24499

 num     #instances         #bytes  class name
----------------------------------------------
   1:           574       35022056  [I
   2:          5281         445528  [C
   3:           395         148520  java.lang.Thread
   4:           499         136648  [B
   5:          4609         110616  java.lang.String
   6:           725          82984  java.lang.Class
   7:           711          45248  [Ljava.lang.Object;
   8:           390          31200  [Ljava.lang.ThreadLocal$ThreadLocalMap$Entry;
   9:           696          27840  java.util.LinkedHashMap$Entry
  10:           862          27584  java.util.HashMap$Node
  11:           219          21024  sun.util.calendar.Gregorian$Date
  12:           387          18576  java.util.concurrent.ThreadPoolExecutor$Worker
  13:            34          17440  [Ljava.util.HashMap$Node;
  14:           400          16000  java.security.AccessControlContext
  15:           225          14400  java.net.URL
```

### 4.2 生成堆转储文件

> **生成堆转储文件有几种方式：主要分为线上生成，远程生成。**

#### 4.2.1 在线生成堆转储文件

**jcmd 命令：**`jcmd <pid> GC.heap_dump /path/filename.hprof` 

**jmap 命令：**`jmap -dump:live,format=b,file=/path/filename.hprof <pid>`，该命令会强制应用程序先进行 `Full GC`，请谨慎只用。

**JVM 参数：**`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/*.hprof`，该命令会在应用程序出现 `OOM` 异常时，在指定路径生产堆转储文件。

#### 4.2.2 远程生成堆转储文件

远程连接的前提，需要使用 [JVM 参数配置 JMX](http://notebook.bonismo.ink/#/Java/JVM/PerformanceTuningGuide?id=_21-jvm-%e4%b8%a4%e7%a7%8d%e9%80%9a%e4%bf%a1%e6%96%b9%e5%bc%8f)

- `Java VisualVM`

![jvisual-heap](https://raw.githubusercontent.com/StayHungryStayFoolish/notebook-img/master/img/jvm/jcmd-hprof.jpg)

- `JDK Mission Control`

- `Eclipse Memory Analyzer Tool(MAT)`

**上述三个工具不但可以查看堆转储、线程转储文件，还可以远程连接直接生成堆转储、线程转储文件。**

- `Jhat`  采用 `html` 方式显示。

#### 4.2.3 其他工具推荐

[Arthas](https://github.com/alibaba/arthas/blob/master/README_CN.md)：**Alibaba** 开源的 Java 诊断工具，必须和应用程序同一服务器，否则无法 attach。

[btace](https://github.com/btraceio/btrace)：需要配置一些环境，不如 Arthas 便捷。

### 4.3 根据堆转储文件分析，解决 OOM

根据堆转储文件分析，可以清晰的发现类实例加载情况，可以根据加载比例比较高的类实例一步步追寻到应用程序的代码中，进而解决 OOM 问题。

