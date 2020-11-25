# JIT 的逃逸分析算法

## 1. Escape Analysis(逃逸分析，简称 EA)

> 一个后缀为 `.java` 的文件通过 `javac` 前端编译组件转为 `.class` 字节码文件。字节码文件通过 `Interpreter(解释器)` 翻译为对应的机器指令，当 JVM 发现部分代码片段在执行时触发了 [JIT 编译策略](http://notebook.bonismo.ink/#/Java/JVM/JDK?id=_232-jitjust-in-time-compiler%e5%8d%b3%e6%97%b6%e7%bc%96%e8%af%91%e5%99%a8) 被称为热点代码，`JIT(即时编译器)`则会把热点代码翻译为本地机器相关的指令，然后进行很多种优化。
>
> 常见的优化如下：
>
> 1.  标量替换
> 2.  内联
> 3.  锁消除
> 4.  锁膨胀
> 5.  空值检查消除
> 6.  类型检查消除

**Hotspot 内部 Stack Memory 中存储对象与数组的情况：**

Java 在 `Stack Memory` 不存储任何对象。它的确在 `Stack Memory` 存储了局部变量：基本类型如 `int` 或 `boolean`。它还将指向对象的指针存储在 `Stack Memory` 中。但是它不会将对象本身存储在其中。创建的任何对象 `new` 始终会在 `Heap Memory` 创建。

数组可以在 `Stack Memory` 分配的条件： JIT 可以确定数组的大小，并且可以确定数组的静态索引值，则可以在 `Stack Memory` 分配。

## 2. 一篇奠定 JDK 优化的论文

在 1999年，一篇名为 [Escape Analysis For Java](https://www.cc.gatech.edu/~harrold/6340/cs6340_fall2009/Readings/choi99escape.pdf) 的论文出现在 [ACM SIGPLAN](https://www.sigplan.org/) 大会上。该论文阐述了一种算法，该算法可以检测一个对象 `O` 是否会从当前之前的方法 `M` 或当前线程 `T` 中逃逸。

该论文为 **JDK6u14** 引入 **HotSpot JVM Escape Analysis** 优化奠定了基础。从 **JDK6u21**，Escape Analysis 成为 **C2（Server） 编译器**的默认配置。

**需要注意的是 EA 从来不是优化，只是分析手段，是为 JVM 的 `Optimization(优化器)` 提供了更好的数据支撑。**

## 3. 本文需要了解的知识点

`Scalar Types(标量数据类型)` 指的是基本数据类型和对象的引用。

`Aggregate Types(聚合数据类型)` 指的是包含任意个数的标量数据类型的一个聚合结构。

`Scalar Replacement(标量替换)`  是将一个对象对字段的访问重新映射成对标量数据的访问的合成操作。该过程是在 **JVM** 的 `Optimization(优化器)` 中进行的。

**发生标量替换的条件是：不得将对象分配给对象（或静态）字段，并且也不得分配给数组，所以一个数组对象不会发生标量替换。**

`Inline(内联)` 指的是通过将最常执行的方法的调用替换为其主体来优化运行时已编译源代码的方法。本质上是 JIT 尝试内联我们经常调用的方法，从而避免了方法调用的开销。以下是内联代码例子：

```java
public int add(int a, int b) { 
  return a + b + 1; 
} 

public void testInline() { 
  int v1 = add(2, 5); 
  int v2 = add(7, 13); 
} 

// 上述代码可以直接优化为一个 testInline()，优化结果如下
public void testInline() { 
  int v1 = 2 + 5 + 1; 
  int v2 = 7 + 13 + 1 
} 
```

## 4. 基于 EA 标识的对象状态

>   **代码可以执行 EA 的前提是满足 JIT 的热点代码要求。具体请参考 [JIT 策略](http://notebook.bonismo.ink/#/Java/JVM/JDK?id=_231-interpreter%e8%a7%a3%e9%87%8a%e5%99%a8)**
>
>   **EA 的基本行为就是分析对象的动态作用域，确定对象是否会在作用域之外被改变指针。**

`JIT` 根据逃逸分析算法最终标识出一个对象三种状态

1.  **NoEscape(没有逃逸)** 定义了方法 `M` 创建了对象 `O` ，该对象 `O` 不会逃逸出该方法 `M`。**即方法内对象在当前方法不可被引用。**
2.  **ArgEscape(参数逃逸)** 定义了一个对象 `O` 作为方法 `M` 的参数被传递或或方法中的参数所引用，但是没有逃逸出当前线程 `T`。**即对象作为参数传递给方法时，在该方法之外或其他线程不可被引用。**该状态是通过被调用方法字节码确定的。
3.  **GlobalEscape(全局逃逸)** 定义了一个对象 `O` 可以被其他方法或线程所引用。**即对象可以逃逸出方法或线程，可以被引用。**有三种情况可以确定为全局逃逸。
    1.  一个对象作为**方法结果**返回
    2.  `static` 修饰的静态字段或者自身就是 `GlobalEscape` 对象。
    3.  使用 `@Override` 重写 `Object` 对象内的 `finalize()` 方法，因为对象最终将对 JVM 的  `finalizer(终结器)` 保持可见。

以上三种状态，除了具有 `GlobalEscape` 标识对象不能被优化外，`NoEscape` 和 `ArgEscape` 标识对象都可以被优化。其中 `ArgEscape` 可以进行 `锁消除` 优化。

### 4.1 EA 标识状态代码演示

```java
public class EscapeAnalysis {
    
    private static Escape escape;

    // GlobalEscape - 方法返回引用，发生指针逃逸
    public Escape methodPointerEscape() {
        return new Escape();
    }
    
    // GlobalEscape - static 发生指针逃逸 
    public void globalVariablePointerEscape() {
        escape = new Escape();
    }
    
    // GlobalEscape - 重写 finalize()，对 JVM 的 finalizer 保持可见，也属于逃逸
    @Override
    protected void finalize() throws Throwable {
        System.out.println("Method finalize is GlobalEscape.");
    }
    
    // GlobalEscape - 方法返回引用，发生指针逃逸，可以使用 toString() 方式改变指针从而避免指针逃逸
    public StringBuffer createString(String... values) {
        StringBuffer stringBuffer = new StringBuffer();
        for (String string : values) {
            stringBuffer.append(string).append(",");
        }
        return stringBuffer;
    }
    
    // NoEscape - 没有发生逃逸，改变了 stringBuilder 的指针并指向了 String 对象
    public String createString(String... values) {
        StringBuffer stringBuffer = new StringBuffer();
        for (String string : values) {
            stringBuffer.append(string).append(",");
        }
        return stringBuffer.toString();
    }

    // ArgEscape - 实例引用发生指针逃逸
    public void instancePassPointerEscape() {
        methodPointerEscape().printClassName(this);
    }
    
    // NoEscape - 没有发生逃逸，对象只在方法内有效
    public void noEscape() {
        Object o = new Object();
    }
}

public class Escape {
    
    // ArgEscape 的 EscapeAnalysis 逃逸到 printClassName() 方法
    public void printClassName(EscapeAnalysis analysis) {
        System.out.println(analysis.getClass().getName());
    }
}
```

以上代码可以通过使用 JVM 参数开启 `-XX:+DoEscapeAnalysis -XX:CompileThreshold=1` 或关闭 `-XX:-DoEscapeAnalysis -XX:CompileThreshold=1`  配合 `jmap -histo pid` 命令进行查看。具体请参考 [JVM 调优参数表](http://notebook.bonismo.ink/#/Java/JVM/PerformanceTuningGuide?id=_41-jvm-%e8%b0%83%e4%bc%98%e5%8f%82%e6%95%b0%e8%a1%a8)

### 4.2 NoEscape 

`NoEscape` 标识对象不总是在 `Stack Memory` 上进行分配，有些情况也会采用 `Scalar Replacement(标量替换)` 优化方式。

此处跟很多资料的阐述不一致，大多文章介绍的观点是 `Stack Allocate` 的本质是 `Scalar Replacement`。经过大量的资料查阅，看到了一些 `OpenJDK` 开发者的邮件内容，所以此处贴出链接及原文。[Stack allocation prototrpe for C2](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-January/036849.html)

**邮件原文摘录：**

>   Hi.
>
>   Reading between the lines, the difference between scalar replacement and what you call "stack allocation" is that the latter creates an oop pointing into the stack frame. Is that correct?
>
>   If so, there is another consideration to take into account. Loom continuations can relocate stack frames in memory, and currently, compiled frames do not contain pointers into the stack. This relocation can potentially happen at any safepoint, but the continuation algorithm has two modes. In the slow mode -- used when there are interpreted frames or, for forced preemption, at any arbitrary safepoint -- the stack frames are parsed and walked, and so pointers in the stack that point into the stack could be patched (I assume, from your description that you do not allow oops outside the stack to point into the stack). In the fast mode, we avoid parsing the stack and we just copy the frames as-is, and it can occur at any nmethod call. If your "stack-oops" can currently live across nmethod calls, how much would this optimization be compromised if you disallowed that (i.e. stack oops to stack-allocated objects would not live across nmethod calls)?
>
>   Also, if my understanding is correct, I'm not sure you need to limit the optimization to -UseCompressedOops. I believe that even when compressed oops are used, stack oops can be wide; i.e. we do find frames with mixed narrow and wide oops in them, and the oopmaps allow for that possibility.
>
>   Ron  
>
>     
>
>   Hi Ron, 
>
>   > Reading between the lines, the difference between scalar replacement and what you call "stack allocation" is that the latter creates an oop pointing into the stack frame. Is that correct? 
>
>   Yes, you are correct. There will be oops on the stack that will point to stack locations in the same nmethod frame. We used the BoxLockNode in C2 and these stack allocated objects end up at the bottom of the frame.
>
>   > Also, if my understanding is correct, I'm not sure you need to limit the optimization to -UseCompressedOops. I believe that even when compressed oops are used, stack oops can be wide; i.e. we do find frames with mixed narrow and wide oops in them, and the oopmaps allow for that possibility.
>
>   Our present limitation was mainly related to autos that can point to either stack allocated object or heap object, depending on the context. With compressed oops enabled we saw the oop always undergo an EncodeP before it was stored on the stack, after a merge point. It would be great if there's a way to overcome this in the IR, we are both new to OpenJDK and could use guidance on how to approach the problem.
>
>   Thanks a lot for your insights and feedback!
>
>   Nikola

#### 4.2.1 Lock Elision(锁消除)

`StringBuffer` 和 `Vector` 都是用 `synchronized` 修饰线程安全的，大部分情况它们的作用域只在当前线程，此时编译器就会进行锁消除优化。

```java
public void lockElision() {
    // NoEscape
    Object o = new Object();
    synchronized(o) 
        System.out.println(o);
    }
}

// 经过逃逸分析后，确定属于 NoEscape，JIT 会将锁优化掉，优化结果如下
public void lockElision() {
 	Object o = new Object();
    System.out.println(o);
}
```

#### 4.2.2 Scalar Replacement(标量替换)

```java
public class Coordinate {
	private int x;
	private int y;
	private int z;
    
    public Coordinate(int x, int y, int z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }
    
    // 省略 setter / getter 
}

public class ScalarReplacement {

    private void printData() {
        // NoEscape
        Coordinate coordinate = new Coordinate(1, 2, 3);
        System.out.println(coordinate.x + " : " + coordinate.y + " : " + coordinate.z);
    }
    
    public static void main(String[] args) {
        ScalarReplacement replacement = new ScalarReplacement();
        replacement.printData();
    }
}

// 经过逃逸分析后，确定属于 NoEscape，JIT 会将进行标量替换，printData() 方法优化结果如下

    private void printData() {
		int x = 1;
		int y = 2;
		int z = 3;        
        System.out.println(x + " : " + y + " : " + z);
    }
```



