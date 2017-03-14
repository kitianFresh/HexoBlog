---
title:
  Java内存管理之GC
date:
  2017-03-14
categories:
 - Java 
 - GC
tags:
 - GC
 - Memory Management
 - Java
---

# 为什么Java要做GC
[Java 设计原则](http://www.oracle.com/technetwork/java/intro-141325.html)
 
 1. **Simple, Object Oriented, and Familiar**
    - 简单，面向对象，语法和 C/C++ 类似 
 2. **Robust and Security**
    - 健壮且安全， 因为 Java 被翻译成字节码之后在虚拟机上解释运行，会经过编译时期的**compiler-time checking，以及运行时期的 run-time checking 双重检查**，因此只要虚拟机足够健壮，不可能发生字节码注入的攻击。
 3. **Architecture Neutral and Portable**
    - 独立于平台且可移植，由于 有 JVM 的支持，Java 语言都会被编译成字节码来执行，**字节码是统一的，并且Java的基本数据类型大小在不同的平台都是一样的，因此可移植性好。**
 4. **High Performance**
    - 高性能，**自动内存管理**，不需要自己手动释放内存，避免了指针的复杂性。Java并不是没有指针，只是是一种受限制的指针。
 5. **Interpreted, Threaded, and Dynamic**
    - **解释型，内置线程支持，编译时静态类型检查，运行时动态链接。**

> Java 和 C/C++ 的区别是什么？
 
 - 编程模型方面
    1. **C语言是面向过程的命令式语言，C++是C语言的超集和扩展，同时支持面向过程和面向对象；而 Java 是纯面向对象的语言，不支持通过面向过程编写单个独立的可运行文件。**
       - 比如我们在设计一个一套排序算法工具类的时候，由于对于排序比较标准是未知的，只能通过外部传递，此时 C/C++ 可以直接支持回调函数，传递函数指针，在外部定义比较函数。
       - Java 由于只支持类和对象，因此 回调的实现必须依附于某一个类，这就是为什么 Java 会出现 comparator 比较器接口，因为Collection 对集合进行排序的时候，由于对象自身未实现 comparable 接口，无法知道比较标准，只能通过外部传递，但是因为JDK1.7以前不支持直接传入函数作为参数，只能借由一个类或者接口来代替函数。
    2. **C++ 支持多继承，Java不支持，但是Java通过实现多接口的方式弥补了这点。**
 
 - 编译执行方面
    3. **C/C++ 是编译型语言，Java是编译+解释型的语言。**
       - C/C++ 编译之后直接是和机器对接的二进制可执行文件，因此，不同机器指令格式会影响最终结果，因此 C/C++ 的可移植性不好，往往需要交叉编译非常复杂。
       - Java 的可移植性很好，因为 Java 程序首先被编译成 字节码文件，然后 JVM 解释执行字节码文件。每一个 Java 程序都是依赖一个 JVM 进程运行起来的。这里的字节码对于 JVM 来说就是他的输入！由于 Java 有一套规范的语言规范，因此只要 compiler 编译之后的 字节码文件符合 JVM 执行规范，就可以实现`一次编译，到处执行`。

 - 内存管理方面
    4. **C/C++ 需要程序员手动维护内存，手动释放不再使用的内存。Java 则由虚拟机的GC来管理内存的自动内存管理技术。** 

 - 线程管理方面
    5. **Java 内置线程和并发包，而 C/C++ 需要依赖于操作系统线程进程。C/C++ 是更接近于底层的语言。**

Java 为了达到以上5点设计原则，必须要实现自动内存管理，因此 GC 必不可少。本质上来讲，**Java 是被程序执行的程序， 而 C/C++ 是直接由机器执行的程序。这也是 Java 有很多特性(比如反射动态的修改代码，动态的创建类等) C/C++ 无法满足的原因。**

# GC如何判定对象死亡
> **为什么 Java 的对象可以自动管理呢？如果程序员不显式的指明某个对象不再使用了，JVM怎么知道对象死亡了？**

对于 C/C++ 显然是不可能实现的，你不可能让硬件直接帮你检查哪些对象不再使用了吧。如果检查就是一种性能的大大降低。有一种叫 [valgrind](http://www.cprogramming.com/debugging/valgrind.html) 的 C/C++ 程序内存泄露检查工具，这个就相当于你把程序放到另一个程序的环境里跑，另一个程序拿到符号表什么之类的程序的元数据信息，就可以知道哪些内存没有释放了。也即是说，只要拿到程序的编译时的很多元数据信息，就容易判断对象是否已经不再使用了。那么对于 Java 能做到也是显而易见的事情了，因为他就是一个程序在根据元数据信息执行另外一个程序！

我们来看一份代码
```java
public class TestAllocation {
	private static final int _1MB = 1024 * 1024;
	/**
	 * VM 参数： -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
	 */
	public static void testAllocation() {
		byte[] a1, a2, a3, a4;
		a1 = new byte[2 * _1MB];
		a2 = new byte[2 * _1MB];
		a3 = new byte[2 * _1MB];
		a4 = new byte[4 * _1MB];
		
		
	}
	
	public static void main(String[] args) throws InterruptedException {
		TestAllocation.testAllocation();
		
	}
}
```

## 可达性分析
> 如何才能知道 a1，a2，a3，a4所指向的对象不再使用呢？

有一个指标就是现在没有指针指向他们！怎么才能知道没有指针指向他们呢？简单的想其实就是找等式左边的指针值还是不是这个对象的地址，如果所有指向该对象的指针都不在了(后者值不是该对象地址了)，那么这个对象就可以认为死亡了。而等式左边的值一定是一个引用即指针，我们只要扫描现存的所有指针就行了，能扫描到的都是存活对象，不能扫描到的都是未存活对象。此时暴力的方法就是扫描一个指针数组里的指针，和另一个对象数组地址是否匹配的过程，O(n^2)。

但是有一个问题就是，这些指针如何知道呢？对象地址容易在申请的时候就知道了，但是左边的指针咋办，因此使用数组不好实现，那么我们需要考虑树和图这种结构了，这个就是可达性分析。

我们知道，所有栈中的引用都会指向某个堆中的对象，而堆中的对象又会有成员指向另一些对象，这样就构成了一个巨大的图。我们需要从很多根节点开始，进行图搜索遍历算法DFS或者BFS。遍历到的节点都是存活的，未遍历到的即认为死亡。这样根节点就必须是那些不能死亡的对象了，我们必须记住他们，这些对象包括

 - **虚拟机栈（栈帧中的本地变量表）中引用的对象。**
 - **方法区中类静态属性引用的对象。**
 - **方法区中常量引用的对象。**
 - **本地方法栈中JNI(即native栈)引用的对象**

GC-roots 根枚举算法

{% asset_img gc-roots.png gc-roots %}

## 引用计数法
引用计数法，但是逻辑很难讲通。说的是对象每次被赋值给某一个引用，计数器就加1，每次被解引用计数器就减1，加1好理解，碰到一个引用被赋值为对象地址，就加1，当引用更换指向，那么该对象的计数器就减去1。但是这个很难理解是怎么做到的！我们只需要知道，**引用计数法的缺点是无法解决循环引用的问题。**

```java
public class ReferenceCountGC {
	public Object instance = null;
	private static final int _1MB = 1024 * 1024;
	private byte[] memory = new byte[4 * _1MB];
	
	public static void testGC() {
		ReferenceCountGC objectA = new ReferenceCountGC();
		ReferenceCountGC objectB = new ReferenceCountGC();
		
		objectA.instance = objectB;
		objectB.instance = objectA;
		
		System.gc();		
	}
	
	public static void main(String[] args) {
		ReferenceCountGC.testGC();
	}
}
```
如果运行以上代码，那么引用计数法就无法回收这两个循环引用的对象了，尽管已经没有人再使用他们。因为他们的计数器是 1。

## finalize 对象死亡逃逸
我们知道 Java 中有 `System.gc()` 显式的调用 gc，但是调用之后不再被引用的对象就一定会被清理吗？我们来看看一个对象自救的代码。

```java
public class FinalizeEscapeGC {
	public static FinalizeEscapeGC SAVE_HOOK = null; // 自救的钩子，如果能够被再次引用，那么对象就会逃逸成功，但只有一次机会，因为 finalize 最多调用一次
	
	public void isAlive() {
		System.out.println("yes, i am still alive. :)");
	}
	
	@Override
	protected void finalize() throws Throwable{
		super.finalize();
		System.out.println("finalize method executed");
		FinalizeEscapeGC.SAVE_HOOK = this;
	}
	
	public static void main(String[] args) throws Throwable{
		SAVE_HOOK = new FinalizeEscapeGC();
		// 第一次进行自救，调用gc，然后gc会 把 覆盖了 finalize 方法且finalize 从未执行过的对象放入 F-QUEUE 中等待执行他们的 finalize 方法，
		SAVE_HOOK = null;
		System.gc();
		// finalize 线程方法优先级较低，暂停一会儿等待他执行
		Thread.sleep(500);
		if (SAVE_HOOK != null) {
			SAVE_HOOK.isAlive();
		}
		else {
			System.out.println("no, i am dead. :(");
		}
		
		SAVE_HOOK = null;
		System.gc();
		// finalize 线程方法优先级较低，暂停一会儿等待他执行
		Thread.sleep(500);
		if (SAVE_HOOK != null) {
			SAVE_HOOK.isAlive();
		}
		else {
			System.out.println("no, i am dead. :(");
		}
		
	}
}

/** 
finalize method executed
yes, i am still alive. :)
no, i am dead. :(
*/
```
我们可以看到，在第一次调用 gc之后，对象竟然活了，没有被清理，但是第二次再次调用的时候，就死亡了！主要是因为 gc 调用以后，并不是立即回收对象，而是检查对象是否需要执行 finalize 方法，需要执行的话，就把他送到 F-Queue 中，然后会通过JVM建立的 finalizer线程扫描队列中的对象，执行对象的finalize方法。而且 finalizer 线程优先级很低，不保证运行一次能够执行完！

**这里判断是否送进F-Queue 的标准就是，对象覆盖了 finalize() 方法且 finalize 从未被虚拟机执行过。如果是没有覆盖或者 finalize 已经被虚拟机调用过了，则直接死亡，但也不一定立即就会被回收！**

# 堆中内存布局和管理
> **既然可以知道对象的死活了，JVM如何布局和管理这些死亡和存活的对象才能提高内存利用率防止内存耗尽呢？即堆中的布局是什么样的呢？**

内存管理最简单的把堆看成一个一维连续数组，然后维护一个空闲链表，就是分配的时候就从空闲链表删除，释放的时候就插入空闲链表，最后为了效率可能还会引入compact紧缩技术。但是Java为什么要分代呢？

## 分代管理技术

为什么要分代管理呢? **原因就在于，对象大部分是短命的，这样如果对整个堆内存进行无区别的管理，短时间内分配的对象过多，整个空间分散着对象，当进行删除紧缩时，耗时巨大，性能就急剧下降。如果把堆分成几块，这样，年轻的对象在较小的空间不断的分配释放，会更快些，对于较少部分的年老对象，年龄达到后就可以放入老年区域，由于是少部分，而且不是频繁的移动，因此性能得以提高！**

{% asset_img hotspot-heap-structure.PNG hotspot-heap-structure %}

 - Young Generation（Heap）
   - eden space
   - survivor space 0
   - survivor sapce 1
 - Old Generation (Heap)
 - Permanent Generation（Method Area）

> 为什么Eden 和 Survivor 的比例一般是 8:1?

因为**一般对象被创建后，存活的少，死亡的多**。大部分对象都是短命的，为了有足够的空间给未来做对象申请，因此 Eden 尽量大些。 

GC触发分类
 - Minor GC 对新生代空间进行清理
 - Major GC 对老年代空间进行清理
 - Full GC 清理整个堆，Young & Old

分代管理 GC 触发过程
 1. Object Allocation，Eden未满，直接分配

{% asset_img object-allocation.PNG object-allocation %}

 2. Filling Eden，此时 minor gc 触发

{% asset_img filling-eden.PNG filling-enden %}

 3. Copy Referenced Objects，会进行标记清理，然后拷贝Eden中幸存对象到 S0

{% asset_img cp1.PNG cp1 %}

 4. Object Aging，再一次 minor gc 触发，Young Generation 中的所有幸存者(包括S0和Eden) 拷贝到 S1.（S0和S1始终有一个为空），并且 S0拷贝来的对象年龄加1

{% asset_img cp2.PNG cp2 %}

 5. Additional Aging，此时，S0和S1身份互换了，S0为空，S1是非空，再一次 minor gc时，又从 Eden 和 S1 拷贝到 S0，其中 S中的老对象年龄加1.

{% asset_img cp3.PNG cp3 %}

 6. Promotion, 年龄反复交替，直到年龄达到阈值，达到阈值的对象才被拷贝到 Old Generation.

{% asset_img promotion.PNG promotion %}

 7. 最后，major GC 执行的时候，会检查


## 垃圾收集算法
 - 标记-清除算法(mark-sweep)
 - 复制算法(half-copy)
 - 标记-整理算法(mark-compact)

***我们知道复制操作是耗费时间的，如果复制量大，性能就大打折扣，因此这个标准直接决定了不同代之间使用的垃圾收集算法。**

 - 新生代： **少量对象存活，适合复制算法** 
 - 老年代： **大量对象存活，适合 MS 和 MC**

# GC 类型
垃圾回收线程/GC线程：垃圾收集器工作时的线程。
应用程序和GC都是一种线程，以Java的main方法为例：应用程序的线程指的是main方法的主线程，GC线程是JVM的内部线程。

在GC过程中，如果GC线程必须暂停应用程序线程（用户线程），则发生Stop the World(卡顿现象)。当然也可以允许GC线程和应用程序线程一起运行，即GC并不会暂停应用程序的线程。

串行、并行、并发：串行和并行指的是垃圾收集器工作时暂停应用程序（发生Stop the World），使用单核CPU（串行）还是多核CPU（并行）。
 - 串行（Serial）：使用单核CPU串行地进行垃圾收集
 - 并行（Parallel）：使用多CPU并行地进行垃圾收集，并行是GC线程有多个，但在运行GC线程时，用户线程是阻塞的
 - 并发（Concurrent）：垃圾收集时不会暂停应用程序线程，大部分阶段用户线程和GC线程都在运行，我们称垃圾收集器和应用程序是并发运行的。

{% asset_img stop-the-world.png stop-the-world %}

> 我们知道 JVM 需要运行我们程序员写的java代码，那么垃圾收集器何时发生，如何发生呢？

JVM 启动之后，Java Main 线程当然是优先级高的县城了，然后在合适的时间就会触发 GC 线程的运行。

JVM新生代GC
 1. Serial Copying：单CPU、新生代小、对暂停时间要求丌高的应用
 2. Parallel Scavenge：多CPU、对暂停时间要求较短的应用
 3. ParNew：Serial Copying的多线程版本

JVM老年代GC
 1. Serial MSC/Serial Old/Serial Mark Sweep Compact
 2. Parallel Compacting/Parallel Old
 3. Concurent Mark Sweep

最新GC 
 G1

> 垃圾收集器如何组合使用，才能让提高 JVM 的性能呢？

{% asset_img JVMGC.png JVMGC %}

# 小结
 1. 对象生命周期规律是大多数对象命短，基于这个规律，垃圾收集器分代，YG & OG， YG中每次少量对象存活，OG中每次大量对象存活。
 2. Eden 比 Survivor 大也是新生对象命短的原因，这样就能留出足够空间给以后使用。
 3. 垃圾回收算法也是看复制开销大小。存活对象越多，复制开销越大。
 4. JVM收集器并行串行并发，并发能够改善响应时间，减少卡顿。

# 参考
 - [Java Garbage Collection Basics](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
 - [Memory Management in the Java HotSpot Virtual Machine](http://www.oracle.com/technetwork/java/javase/memorymanagement-whitepaper-150215.pdf)
 - [HotspotOverview](https://www.cs.princeton.edu/picasso/mats/HotspotOverview.pdf)
 - [understanding-java-garbage-collection](http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/)
 - [java-garbage-collection-distilled](https://mechanical-sympathy.blogspot.com/2013/07/java-garbage-collection-distilled.html)
 - [JVM GC](http://zqhxuyuan.github.io/2016/07/26/JVM/#)这篇博客基本涵盖所有，非常详细！
