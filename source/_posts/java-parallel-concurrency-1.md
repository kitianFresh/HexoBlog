---
title:
  从数组求和谈Java并行程序设计(一)
date:
  2017-05-05
categories:
- Java
tags:
- Java
- Parallel
- Concurrency
- JVM
- Heap
- Stack
- Divide and conquer
- Fork Join
- Start Run
- Multi-Thread
---


大家都知道如何串行的计算一个数组的和, 而对于现代多CPU计算机甚至是分布式系统而言, 采用串行方式大大浪费了计算资源, 而且当数据规模达到一定程度的时候, 串行算法往往无法胜任高响应时间的要求. 本文从几个小实验开始, 慢慢探讨 Java 并行程序设计方案. 

# Sum An Array By Multi-Threads
实验环境 `sudo lshw`:
```
*-cpus
        Architecture:          x86_64
        CPU op-mode(s):        32-bit, 64-bit
        Byte Order:            Little Endian
        CPU(s):                4
        On-line CPU(s) list:   0-3
        Thread(s) per core:    1
        Core(s) per socket:    4

*-memory
        description: System Memory
        physical id: 7
        slot: System board or motherboard
        size: 8GiB

```
数组求和实验, 采用串行和多线程的形式进行比较. 实验的测试代码:
```Java
	public static void main(String[] args) {
		int size = 400000;
		int[] a = new int[size];
		for (int i = 0; i < size; i++) {
			a[i] = i;
		}
		
		int sum = 0;
		double t = 0;
		long start, end;
		int loops = 5;
		int cpus = Runtime.getRuntime().availableProcessors();
		int[] threads = new int[]{cpus/4, cpus/2, cpus, cpus + 2, 2 * cpus, 2*cpus+2, 3*cpus, 3*cpus + 2, 4*cpus};
		double[] ts = new double[loops];
		String str0 = String.format("%20s\t%15d\t%15d\t%15d\t%15d\t%15d\t%15s\n", "Threads\\Loops",1,2,3,4,5,"Average(ms)");
		System.out.println(str0);
		try {
			for (Integer cores : threads) {
				t = 0;
				for (int i = 0; i < loops; i++) {
					start = System.currentTimeMillis();
					sum = ArraySumer.sum(a, cores);
					end = System.currentTimeMillis();
					ts[i] = end - start;
					t += ts[i];
					
				}
				t = t / loops;
				String str1 = String.format("%20s\t%15f\t%15f\t%15f\t%15f\t%15f\t%15f\n", cores+"", ts[0], ts[1], ts[2], ts[3], ts[4], t);
				System.out.println(str1);
			}
			
			
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
```

## one thread (sequential)
单线程就是一种串行求和, 设置 threadSize = 1即可!

## different thread nums compares (parallel)
本实验**对每一种线程个数(Threads)做了五次实验(Loops=5), 然后求出五次的平均性能(Average).**

|Threads\Loops|              1|	              2|	            3|	             4|	             5|	    Average(ms)|
|------------:|--------------:|---------------:|----------------:|---------------:|--------------:|---------------:|
|    1	      | 4.000000      |   5.000000     |  0.000000       |     1.000000	  | 0.000000      |   2.000000     |
|    2	      | 0.000000      |   0.000000     |  1.000000       |     0.000000	  | 0.000000      |   0.200000     |
|    4	      | 0.000000      |   0.000000     |  1.000000       |     0.000000	  | 0.000000      |   0.200000     |
|    6	      | 0.000000      |   1.000000     |  0.000000       |     1.000000	  | 1.000000      |   0.600000     |
|    8	      | 1.000000      |   0.000000     |  1.000000       |     0.000000	  | 1.000000      |   0.600000     |
|   10	      | 1.000000      |   1.000000     |  0.000000       |     1.000000	  | 5.000000      |   1.600000     |
|   12	      |10.000000      |   3.000000     |  1.000000       |     0.000000	  | 1.000000      |   3.000000     |
|   14	      | 1.000000      |   1.000000     |  0.000000       |     1.000000	  | 0.000000      |   0.600000     |
|   16	      | 1.000000      |   1.000000     |  8.000000       |     1.000000	  | 0.000000      |   2.200000     |

从以上实验结果可以看出, **对于多线程求和, 线程数目并不是越多性能越好, 一般设置为可用处理器个数或者可用处理器个数的2倍性能最佳!**

## shared and local memory in my code
以下是实验中使用的  `ArraySumer` 类的代码.
```java
public class ArraySumer extends Thread{
	public int[] a;
	public int low;
	public int high;
	public int result = 0;
    public void run() {
		for (int i = this.low; i <= this.high; i++) {
			result += a[i];
		}
	}
	public static int sum(int[] arr, int threadSize) throws InterruptedException {
		int len = arr.length;
		int result = 0;
		ArraySumer[] as = new ArraySumer[threadSize];
		for (int i = 0; i < threadSize; i++) {
			as[i] = new ArraySumer(arr, i*len/threadSize, (i+1)*len/threadSize-1);
			as[i].start();
		}
		for (int i = 0; i < threadSize; i++) {
			as[i].join();
			result += as[i].result;
		}
		return result;
		
	}
	
}
```
从代码中可以看出, 所有线程共享数组变量 `a`. 主线程和从线程都只对 `a` 进行读取操作. 主线程通过局部变量的形式将参数 `low`, `high` 传递给从线程, 从线程会写入自己的成员变量 `result`, 而主线程会读取所有从线程的成员变量 `result`. 因此 race condition 可能会发生在这里, 但是这个数组求和程序, 由于从线程的 `as[i].join()` 操作, 主线程会等待对应的从线程都做完(写完 result)之后才开始读取, 因此并没有数据竞争, 是线程安全的.

## performance analysis
**对于多线程 parallel, 并不是选择越多的线程越好, 因为创建线程本身也是耗费资源(时间和空间资源)的.** 极端的例子, 比如现在有处理器P个,P >> n, 我们对于每一个数,都创建一个线程读取, 然后让主线程合并, 这反而没有串行执行的快. 

**因此要想使得多线程 performance 达到最大, 需要选取合适的 threadSize. 从以上的实验可以得出一个经验性的结论就是, 线程数目和当前可得处理器的个数有关系, 一般我们选取cpus或者 2\*cpus比较合理.**

## a better idea for parallel sum array
### amdahl's law
假设 $S$ 为系统不可加速部分的比例(这里就是只能串行执行的任务的比例), $n$ 为可加速部分(这里就是并行部分)加速倍数, 那么系统的加速比为
$$
S_e = \frac{1}{S+\frac{1-S}{n}}
$$
以上公式其实是根据定义式子 $S\_e=\frac{T\_{origin}}{T\_{new}}$ 推到而来的.

以上的数组求和程序, 是按照线性的思路拆分任务, 然后得到一个线性的加速比(假设处理器核数是无穷个的话). 即假设原来的串行时间为 $T$. 那么现在将任务均分到 $N$ 个线程上之后, 得到并行时间应该是 $\frac{T}{N}$. 根据加速比的定义 $S\_e=\frac{T\_{origin}}{T\_{new}}$, 这里其实整个任务都是可并行的, 没有串行部分, 得到加速比就是和线程数目程线性关系即 $S\_e=\frac{T}{\frac{T}{N}}=N$.

但是实际往往是处理器个数有限为 $P$ 个, 如果 $N>P$, 线性拆分后的运行时间应该是$\frac{N}{P}\frac{T}{N}=\frac{T}{P}$. 因为第一批线程 $\frac{N}{P}$ 个, 并行的运行在 $P$ 个处理器上, 后来继续, 共有 $\frac{N}{P}$ 批次. 你也可以这样理解, 对于每一个处理器而言, 运行完成一个完整的串行数组求和需要 $T$. 现在 $P$ 个处理器运行完这个数组求和, 均分在每一个处理器上的时间就是$\frac{T}{P}$.

### divide and conquer algorithm for paralleling sum array
采用分治法计算数组和, 分治思想: 主线程将任务分解成两个子任务, 左子线程计算左半边数组和, 右子线程计算右半边数组和, 当左右任务完成之后, 主线程合并左右结果(左右和加起来), 左右线程进行同样的递归分解操作, 分解终止的条件由 `SEQUENTIAL_CUTOFF` 来控制. 核心代码如下:
```java
if (this.high - this.low < SEQUENTIAL_CUTOFF) { // 只剩 SEQUENTIAL_CUTOFF 个的时候, 就直接串行的解决
	for (int i = this.low; i <= this.high; i++) {
		this.result += a[i];
	}
}
else {// 还可以继续分解
	int mid = (this.low + this.high) / 2;
	ArraySumerDc left = new ArraySumerDc(this.a, this.low, mid, SEQUENTIAL_CUTOFF);
	ArraySumerDc right = new ArraySumerDc(this.a, mid + 1, this.high, SEQUENTIAL_CUTOFF);
	left.start();
	count.incrementAndGet(); // 记录创建的线程的个数
	right.start();
	count.incrementAndGet();
//			right.run();
	try {
		left.join();
		right.join();
	} catch (InterruptedException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
	this.result = left.result + right.result;
}

// 测试代码
// size = 400000, 树的高度为 log400000 = 18, 线程个数即这棵平衡二叉树的节点数目, 2^19 = 524288... `Exception in thread "Thread-5839" java.lang.OutOfMemoryError: unable to create new native thread`
// 显然, 这么创建线程来解决这个求和问题是为了并行而并行.JVM的线程个数是有限制的. 即使调整为 40000, log40000 = 15, 2^16 = 65536都是线程数目比数组规模还大.
// size = 4000, log4000 = 12. 2^13 = 8096 个线程, 这个时候 JVM 没有发生 outofmemory error.
// 很简单, 因为sequential_cutoff的条件是每一个线程处理一个数, 即每个叶子处理一个数, 而中间的非叶子节点都在等待左右孩子做完后在相加.
//		System.out.println(5^2);// 注意这个是 异或....
int size = 1<<29; // 线程个数超过 1<<16 开始outofmemory
int cutoff = 1<<25; // h = log(size/cutoff) 为树高. 线程个数 2^(h+1) - 1
int h = (int) (Math.log(size/cutoff)/Math.log(2));
System.out.println("Predict Threads: " + (Math.pow(2, h+1) - 1));
int[] a = new int[size];
for (int i = 0; i < size; i++) {
	a[i] = i;
}
int sum = 0;
long start, end;

start = System.currentTimeMillis();
for (int i = 0; i < size; i++) {
	sum += a[i];
}
end = System.currentTimeMillis();
System.out.println("Sum: " + sum);
System.out.println(end - start + "ms");

sum = 0;
start = System.currentTimeMillis();
sum = ArraySumerDc.sum(a, cutoff);
end = System.currentTimeMillis();
System.out.println("Sum: " + sum);
System.out.println(end-start + "ms");
System.out.println("Real Threads: " + ArraySumerDc.count);
```
经过测试后发现,效果并不是很好, 可能原因是和数组规模以及处理器个数有关系, 申请了 1<<29 个int 数组,共占内存2G, 效果只是和 线性分解相当, <font color="red">原因应该是处理器个数太少的缘故</font>, 因为**分治法创建的线程数目和树的高度有关系, 如果处理器个数是无限的话, 显然分治法更胜一筹, 因为他的时间是 O(lgn), 而线性分解是 O(n/N), 当 n 非常大的时候, N 虽然也可以很大,但是从数量级来看, 显然对数时间更少.**

# A parallel DC Algorithm for computing the max number of an array
采用分治法计算数组的最大(最小)数, 分治思想: 主线程将任务分解成两个子任务, 左子线程寻找左半边数组的最大值, 右子线程寻找右半边数组的最大值, 当左右任务完成之后, 主线程合并左右结果(比较左右结果的较大者返回), 左右线程进行同样的递归分解操作, 分解终止的条件由 `SEQUENTIAL_CUTOFF` 来控制.

## start/run method
根据分治思想, 代码如下:
```java
public class MaxNumber extends Thread{
	int low;
	int high;
	int max = 0;
	int[] numbers;
	int SEQUENTIAL_CUTOFF;
	MaxNumber(int[] numbers, int low, int high, int cutoff) {
		this.numbers = numbers;
		this.low = low;
		this.high = high;
		this.SEQUENTIAL_CUTOFF = cutoff;
	}
	
	public void run() {
		if (high - low < SEQUENTIAL_CUTOFF) {
			for (int i = low; i <= high; i++) {
				max = Math.max(max, numbers[i]);
			}
		}
		else {
			int mid = (low + high) / 2;
			MaxNumber left = new MaxNumber(numbers, low, mid, SEQUENTIAL_CUTOFF);
			MaxNumber right = new MaxNumber(numbers, mid+1, high, SEQUENTIAL_CUTOFF);
			left.start();
			right.start();
			
			try {
				left.join();
				right.join();
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			max = Math.max(left.max, right.max);
		}
	}
	
	public static int max(int[] numbers, int cutoff) {
		MaxNumber mn = new MaxNumber(numbers, 0, numbers.length-1,cutoff);
		mn.run();
		return mn.max;
	}
}
```
**采用覆盖 `run()` 方法的缺陷就是无法返回一个值, 只能通过成员变量的形式存储.**另外, **由于每一次递归创建子任务的时候, 主线程没有做任何计算任务,** 必须要等待左右都完成之后,才开始计算, 这是一种资源的浪费, 因此**一种形式的优化是,让主线程来做右子树的任务!** <font color="red">这样线程数目可以减少一半!</font>
```java
int mid = (low + high) / 2;
MaxNumber left = new MaxNumber(numbers, low, mid, SEQUENTIAL_CUTOFF);
MaxNumber right = new MaxNumber(numbers, mid+1, high, SEQUENTIAL_CUTOFF);
left.start();
//right.start();
right.run();

try {
	left.join();
	//right.join();
} catch (InterruptedException e) {
	// TODO Auto-generated catch block
	e.printStackTrace();
}
```
## Fork/Join framework
以下是改用fork/join framework 的代码:
```java
public class MaxNumberForkJoin extends RecursiveTask {
	int low;
	int high;
	int[] numbers;
	int SEQUENTIAL_CUTOFF;
	
	MaxNumberForkJoin(int[] numbers, int low, int high, int cutoff) {
		this.numbers = numbers;
		this.low = low;
		this.high = high;
		this.SEQUENTIAL_CUTOFF = cutoff;
	}
	
	@Override
	protected Integer compute() {
		int max = 0;
		if (high - low < SEQUENTIAL_CUTOFF) {
			for (int i = low; i <= high; i++) {
				max = Math.max(max, numbers[i]);
			}
			return max;
		}
		else {
			int mid = (low + high) / 2;
			MaxNumberForkJoin left = new MaxNumberForkJoin(numbers, low, mid, SEQUENTIAL_CUTOFF);
			MaxNumberForkJoin right = new MaxNumberForkJoin(numbers, mid+1, high, SEQUENTIAL_CUTOFF);
			left.fork();
			int rightMax = right.compute();
			int leftMax = (int) left.join();
			max = Math.max(rightMax, leftMax);
			return max;
		}
	}

	static final ForkJoinPool fjPool = new ForkJoinPool();
	public static int max(int[] numbers, int cutoff) {
		return fjPool.invoke(new MaxNumberForkJoin(numbers, 0, numbers.length-1, cutoff));
	}
	
}
```
## sequential checker
以下是串行运算的部分代码:
```
for (int i = 0; i < size; i++) {
    max = Math.max(max, numbers[i]);
}
```

## performance comparison
这里, 我设置的 `SEQUENTIAL_CUTOFF` 的值是 2048; 还是一样, 对于每一种情况(Serialize Start/Run 和 Fork/Join) 都采用五次测试取平均来比较平均时间性能. 数组规模是 2^25 次方.

|Pattern\Loops           |	            1|	             2|              3|              4|	              5|    Average(ms)|
|-----------------------:|--------------:|---------------:|--------------:|--------------:|---------------:|--------------:|
|Serialize Baseline	     |   29.673285	 |    30.963514	  |    26.244178  |      26.037493|	      25.921920|      27.768078|
|Parallelize Start/Run	 |13103.625879	 |  9753.860262	  |  9774.133012  |   10805.342403|	    9349.867925|   10557.365896|
|Parallelize Fork/Join	 |   34.131773	 |    20.526604	  |    15.806599  |      10.757907|	      10.060238|      18.256624|

对于2^26 次方的时候, Start/Run 模式会出现 <font color="red">`Exception in thread "Thread-28661" java.lang.OutOfMemoryError: unable to create new native thread`</font> 这种 `OutOfMemory` 的错误. 但是 Fork/Join 可以轻松应对! 

**猜测 Fork/Join framework 内部实现了线程池, 因此可以保证不会因为创建过多线程而导致 JVM 爆出内存错误. 因为每一个线程都有私有线程栈, 线程创建的越多, 需要的内存就越多, 当内存不够的时候, 就无法再创建线程了! **

这里**要么设置更小的线程栈以争取创建更多线程的可能, 要么扩充JVM内存.** 但是栈大小本身 JVM 也是有限制的, 设置过小, JVM 就无法启动而报错!
```
The stack size specified is too small, Specify at least 228k
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

**但是当线程数数目达到一定程度的时候, 创建线程的开销反而大于任务本身了, 这是适得其反的! 另外, 线程栈过小的话, 如果有递归, 这里分治法就是递归, 压栈就不可能很深, 就会限制递归的深度, 限制递归的深度就会限制能解决问题的规模数目!**

去掉 Start/Run 方法之后, 我们设置数组规模到JVM堆可以容纳的最大值, **运行的时候你可以设置JVM 的运行参数 `-Xmx4096M -Xms2000M -Xmn500M`, 因为JVM默认最大堆内存是机器物理内存的 1/4, 而我机器内存8G, 也就是最大是2G, 但是JVM最大堆内存的限制实际上取决于操作系统和物理内存空间, 我的机器最大堆内存可以设置到8G. **

                 Pattern\Loops	              1	              2	              3	              4	              5	    Average(ms)
            Serialize Baseline	     188.076303	     206.252412	     208.105703	     203.449220	     203.056148	     201.787957
         Parallelize Fork/Join	     145.812311	      85.910176	      85.791367	      90.719006	      81.187196	      97.884011

但是**你最好不要设置成最大内存 8G, 除非你的服务器只运行一个 JAVA 程序, 如果你设置成最大物理内存8 G, 然后申请 1<<30 个int整形数组,也即是占用 4G 内存, 然后运行程序,机器会非常卡, 因为处理器和内存资源全部都给 JAVA 程序了, 其他进程就无法很愉快的运行和响应了! 如果设置的 `SEQUENTIAL_CUTOFF` 和数组规模以及处理器个数不太符合的话,由于线程数目过多, Fork/Join 的性能就比不上 Serialize 了.** `SEQUENTIAL_CUTOFF = 1<<11`, 将得到一下结果;

                 Pattern\Loops	              1	              2	              3	              4	              5	    Average(ms)
            Serialize Baseline	     952.887980	    1345.387388	     777.596211	     773.802849	     774.879831	     924.910852
         Parallelize Fork/Join	    3157.547799	    1058.703371	    2999.615653	     274.054247	     245.031633	    1546.990541

因此, 我再次提升 `SEQUENTIAL_CUTOFF = 1<<15`, 此时 Fork/Join 的性能将会更好.

                 Pattern\Loops	              1	              2	              3	              4	              5	    Average(ms)
            Serialize Baseline	     781.488606	     794.344170	     761.915751	     750.881256	     746.211944	     766.968345
         Parallelize Fork/Join	     398.081041	     257.000682	     215.312012	     228.940126	     233.489181	     266.564608


# PrimeNumber By Concurrency And Parallel
找出10亿个数中所有的质数, 由于所有质数寻找之间没有依赖关系, 因此我们很容易想到将这10亿个数均分成N段, 每一段使用一个线程并行计算. 但是这里有一点和数组求和不同, 就是**负载均衡(Load Balancing)问题.** 

在数组求和中, 均匀划分后, 每一段计算时间其实相当的, 但是在**计算质数中不是的, 数越大, 判断这个数是否是质数的时间就越长, 因此均匀划分之后, 其实算完所有的就不是一个真实的线性加速了**. **分配到后面大数部分的线程会运行更久的时间,这个时候其他线程都已经运算完毕,但是这个线程可能还在运行, 这样就无法再充分利用剩余已经闲置的CPU了,从而时间还是取决于最慢的那个.**

那么, 还有没有其他的方案呢? 这个任务是否可以做成 concurrency 的? 可以设计出另外一种方案, 就是**多个线程从一个数据池子里面取出数据来判断质数, 取走一个, 池子的数就减少一个, 这样通过共享资源互斥枷锁的方案进行并发求解**, 是否可以比 parallel 的方案更快呢? 其实是有可能的, 因为你**从 CPU 的角度来讲, CPU 在整个任务求解过程中就会马不停蹄(这正是我们想要的), 不会出现CPU空闲的可能, 这样充分利用了多核处理器资源.**

那么到底哪种方案更好呢? 我们做实验看看吧.
## Parallel
```java
public class PrimeNumberParallel extends Thread{
	int low;
	int high;
	List<Long> primes; // 使用 vector 线程安全的
	
	PrimeNumberParallel(List<Long> primes, int low, int high) {
		this.primes = primes;
		this.low = low;
		this.high = high;
	}
	
	public void run() {
		
		for (long i = low; i <= high; i++) {
			if (isPrime(i)) {
				primes.add(i);
//				System.out.println(i);
			}
		}
	}
	
	public static boolean isPrime(long number) {
		if (number <= 1) return false;
		for (long i = 2; i <= (long)Math.sqrt(number); i++) {
			if (number % i == 0) return false;
		}
		return true;
	}
	
	public static List<Long> primeNumbers(int threads, int size) throws InterruptedException {
		List<Long> primes = new Vector<Long>();
		PrimeNumberParallel[] pns = new PrimeNumberParallel[threads];
		for (int i = 0; i < threads; i ++) {
			pns[i] = new PrimeNumberParallel(primes, i*size/threads, (i+1)*size/threads);
			pns[i].start();
		}
		
		for (int i = 0; i < threads; i ++) {
			pns[i].join();
		}
		return primes;
	}
}

```

## Concurrency
```java
// concurrency for prime
public class PrimeNumberConcurrency extends Thread {
	
	Counter counter;
	long SIZE = 1000;
	List<Long> primes; // 使用 vector 线程安全的
	
	PrimeNumberConcurrency(List<Long> primes, Counter counter, int size) {
		this.counter = counter;
		this.primes = primes;
		this.SIZE = size;
	}
	
	public void run() {
		long number;
		while ((number = counter.getAndIncrement()) <= SIZE) {
			if (isPrime(number)) {
				primes.add(number);
//				System.out.println(number);
			}
		}
	}
	
	public static boolean isPrime(long number) {
		if (number <= 1) return false;
		for (long i = 2; i <= (long)Math.sqrt(number); i++) {
			if (number % i == 0) return false;
		}
		return true;
	}
	
	public static List<Long> primeNumbers(Counter counter, int threads, int size) throws InterruptedException {
		List<Long> primes = new Vector<Long>();
		PrimeNumberConcurrency[] pns = new PrimeNumberConcurrency[threads];
		for (int i = 0; i < threads; i ++) {
			pns[i] = new PrimeNumberConcurrency(primes, counter, size);
			pns[i].start();
		}
		
		for (int i = 0; i < threads; i ++) {
			pns[i].join();
		}
		return primes;
	}
}
```
## PrimeNumber concurrency and parallel performance comparison

# Summary
 1. 对于多线程 parallel, <font color="red">并不是选择越多的线程越好, 因为创建线程本身也是耗费资源(时间和空间资源)的.通过设置合适剪枝策略 `SEQUENTIAL_CUFOFF` 和 与处理器相符的线程数目, 来取得更好的 performance.</font>
 2. 当线程数数目达到一定程度的时候, 创建线程的开销反而大于任务本身了, 这是适得其反的! 另外, <font color="red">线程栈过小的话, 如果有递归, 这里分治法就是递归, 压栈就不可能很深, 就会限制递归的深度, 限制递归的深度就会限制能解决问题的规模数目!Java并发程序选择合适的JVM参数非常重要.</font>
 3. <font color="red">Fork/Join framework 内部实现了线程池, 因此可以保证不会因为创建过多线程而导致 JVM 爆出内存错误.</font> Fork/Join framework 的性能比单纯的 start/run 高很多.内部采用了**工作窃取算法，可以完成更多任务的工作线程可以从其它线程中窃取任务来执行.**
 4. Parallel 还是 Concurrency 需要我们<font color="red">考虑线性分解的 Load Balancing.</font>

# References
 - [JVM堆内存和非堆内存](http://xstarcd.github.io/wiki/Java/JVM_Heap_Non-heap.html)