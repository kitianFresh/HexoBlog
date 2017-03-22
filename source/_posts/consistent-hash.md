---
title:
  一致性Hash

date:
  2017-03-20

categories:
 - DistributedSystem
tags:
 - ConsistentHash
 - Distributed
 - Java
---

# 一致性Hash在分布式中的研究和应用

## 问题起源
CSAPP中对计算机体系的整个存储体系结构做了很好的阐述，目前的计算机也是以存储为中心来设计的，因为处理器的速度以及发展速度远远大于存储器，以至于整个计算机系统的性能瓶颈转移到了存储器上。计算机从早期的处理器为中心发展到以存储器为中心的设计。

**由于存储器的容量与速度和价格成反比，为了使整个系统的性价比达到最大，设计出多级存储层次结构，缓存的概念也由此产生。如果数据在后一级，那么一般会选择将一部分数据缓存在前一级以提高存储性能！**

{% asset_img 存储器层次结构.png 存储器层次结构%}

为了加快访问速度，一般情况下，我们会把经常访问的对象放入内存中，而不是每一次都从后一级层次存储提取出来访问。

> 当内存不够了呢？

方案一是使用 LRU， 也就是我们经常听说的 替换算法 中的一种，最近最少使用算法！这是限于单台机器的时候，如果我们的请求量非常大，那么单台机器不足以胜任，即使有比较好的LRU算法，还是会发生很多不命中的情况，导致替换频繁！

剩下的唯一办法就是扩大内存，但是单台机器的内存也是有限制的，于是我们产生了分布式，就是把本来一台机器不够放的缓存拆分到不同的机器上，这样能够缓存的内容就变多了，以上内存不足，频繁换页的问题也就解决了，即克服了在大数据量下单台机器的扩展性不足以及高效的 LRU 也不管用的缺陷！

但是，如何解决这些缓存的分配任务呢？对于某一个查询对象，我们必须很快的确定(最好是计算出来O(1))他被缓存在哪一台机器上面，回顾数据结构与算法相关知识我们知道， 基于计算的 hash 可以达到这种效果！


## Hash 函数
**Hash 就是将一个无限范围的域值，映射到有限范围的域值。而且实现基于计算的查找i性能，平均时间复杂度是O(1)，而好的hash函数能够保证均匀性和避免碰撞！**

> 如何在分布式应用中使用 Hash 函数？

我们先使用 除模取余的方法，来设计一个 分布式 hash 函数！即 hash(key) = key % n;除模取余虽然简单，但是有个缺点，扩展性很差，如果我们增加一个节点，此时 hash(key) = key % (n+1), 那么大部分key原来的hash就会重新分配的到新值，导致原来的所有缓存都失效，性能就下降了！




## 一致性 Hash
###  概念
一致性 hash 实现了**扩展或者删除一台机器之后，缓存不会大范围改动的问题！** 原来主要是除模取余的方法，每次改变 cache（node） 的数目，都必须该改变模数，导致原来对象的hash值大范围的重新分布，如果我们能取一个固定的模就好了！一致性Hash 的模数一般就取的是 2^31

**一致性hash就是取了一个固定的模，而且这个模足够大，他的巧妙之处是将 原本 映射到 单一点 的对象， 改成映射到某一个区间，然后寻找顺时针(clockwise)最近的 cache（node）。**

### 基本构造 Hash 环

{% asset_img 一致性hash基本概念.png 一致性hash基本概念 %}

### 删除一个节点

{% asset_img 一致性hash节点删除.png 一致性hash节点删除 %}

### 增加一个节点

{% asset_img 一致性hash节点增加.png 一致性hash节点增加 %}

### 初始不均衡

{% asset_img 一致性hash不均衡.png 一致性hash不均衡 %}

### 虚拟节点

{% asset_img 一致性hash虚拟节点.png 一致性hash虚拟节点 %}


**由于每隔 cache 占据一段区间而不是一个点，并且模数固定，因此 一致性 hash 在节点增加或者删除的时候最多影响一个节点或者一段区间上的查询！**


## 一致性 Hash 的实现
我们可以设计出一个 hash 表，并且使用 链地址法 解决冲突，但是 如何才能找到距离 hash(objectKey) 最近的 Node 呢？ 必须要往前遍历这个 hash 的槽位，虚拟节点实际是不存在的，存在的只是他的 key 即 hash 值会在环上拥有一个位置，但是她对应的 value 实际上都是同一个真实的节点，即 value 是指向同一个值。

### 选择数据结构
既需要 hash， 有需要能够遍历且顺序不变，那么 Java 中的 TreeMap 再好不过了。TreeMap 是支持有序遍历的 HashMap。并且 他还有一个支持返回大于等于某个 key 的集合的 tailMap 方法。

### 选择 Hash 函数
hash 函数一定要尽量的产生均衡的分布，注意一致性 hash 的模数 是固定的, 一般就取 int 类型的最大值 + 1，因此我们一般只需要的到一个整数hash即可！如果直接使用 hashCode，由于是内存地址，往往出现地址比较靠近的现象，这里采用 MD5.

### Java 代码
ConsistentHash.java
```Java
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Collection;
import java.util.SortedMap;
import java.util.TreeMap;

class Node {

	String ip;

	public Node(String ip) {
		this.ip = ip;
	}

	public String toString() {
		return this.ip;
	}
}

class MD5Hash implements HashFunction {

	public static String getMD5(String input) {
		try {
			MessageDigest md = MessageDigest.getInstance("MD5");
			md.update(input.getBytes());

			byte byteData[] = md.digest();

			// convert the byte to hex format method 1
			StringBuffer sb = new StringBuffer();
			for (int i = 0; i < byteData.length; i++) {
				sb.append(Integer.toString((byteData[i] & 0xff) + 0x100, 16).substring(1));
			}
			return sb.toString();
			
		} catch (NoSuchAlgorithmException e) {
			throw new RuntimeException(e);
		}
	}

	public static int hashStringToInt(String s) throws NoSuchAlgorithmException {
		// 第一种方法
		// int h = 0;
		// for (int i = 0; i < s.length(); i++) {
		// h = (h << 5) | (h >>> 27); // 5-bit cyclic shift of the running sum
		// h += (int) s.charAt(i); // add in next character
		// }
		// return h;
		
		// 第二种方法
		 int hashValue = 0;
		 for (int Pos = 0; Pos < s.length(); Pos++) { // use all elements
		 hashValue = (hashValue << 4) + s.charAt(Pos); // shift/mix
		 int hiBits = hashValue & 0xF0000000; // get high nybble
		 if (hiBits != 0)
		 hashValue ^= hiBits >> 24; // xor high nybble with second nybble
		 hashValue &= ~hiBits; // clear high nybble
		 }
		 return hashValue;

		 // 第三种方法
//		 int h = -7;
//		
//		 h ^= s.hashCode();
//		
//		 // This function ensures that hashCodes that differ only by
//		 // constant multiples at each bit position have a bounded
//		 // number of collisions (approximately 8 at default load factor).
//		 h ^= (h >>> 20) ^ (h >>> 12);
//		 return h ^ (h >>> 7) ^ (h >>> 4);
	}

	@Override
	public int hash(Object key) {

		try {
			return hashStringToInt(getMD5(key.toString()));
		} catch (NoSuchAlgorithmException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return 0;
	}

}

public class ConsistentHash<T> {
	private final HashFunction hashFunc;
	private final int numberOfReplicas;
	private final SortedMap<Integer, T> circle = new TreeMap<Integer, T>();

	public ConsistentHash(HashFunction hashFunc, int numberOfReplicas, Collection<T> nodes) {
		this.hashFunc = hashFunc;
		this.numberOfReplicas = numberOfReplicas;
		for (T node : nodes) {
			this.add(node);
		}
	}

	/**
	 * 
	 * @param node
	 *            增加一个 cache 节点
	 */

	public void add(T node) {
		for (int i = 0; i < numberOfReplicas; i++) {
			circle.put(hashFunc.hash(node.toString() + "#" + i), node);
		}
	}

	/**
	 * 
	 * @param node
	 *            删除一个 cache 节点
	 */
	public void remove(T node) {
		for (int i = 0; i < numberOfReplicas; i++) {
			circle.remove(hashFunc.hash(node.toString() + "#" + i));
		}
	}

	/**
	 * 
	 * @param key
	 *            对象 key
	 * @return 返回 key 所在的 Node
	 */

	public T get(Object key) {
		if (circle.isEmpty())
			return null;
		int hash = hashFunc.hash(key);

		if (!circle.containsKey(hash)) {
			SortedMap<Integer, T> tailMap = circle.tailMap(hash);
			hash = tailMap.isEmpty() ? circle.firstKey() : tailMap.firstKey();
		}

		return circle.get(hash);
	}

	/**
	 * 工具函数
	 * 
	 * @param node
	 * @param replicaNumber
	 * @return
	 */
	public String getNodeHashKey(T node, int replicaNumber) {
		if (replicaNumber > numberOfReplicas)
			throw new IllegalArgumentException();

		return node.toString() + "#" + replicaNumber;
	}

	/**
	 * 工具函数
	 * 
	 * @param key
	 * @return
	 */
	public int getNodeHash(String key) {
		return hashFunc.hash(key);
	}

}
```

HashFunction.java
```java
public interface HashFunction {
	public int hash(Object key);
}
```
test
```java
@org.junit.Test
public void testConsistentHash() {
  List<Node> nodes = new ArrayList<Node>();
  nodes.add(new Node("10.11.125.33"));
  nodes.add(new Node("10.11.125.34"));
  nodes.add(new Node("10.11.125.35"));
  nodes.add(new Node("10.11.125.36"));
  MD5Hash md5Hash = new MD5Hash();
  int replicas = 2;
  ConsistentHash ch = new ConsistentHash(md5Hash, 2, nodes);
  for (Node node : nodes) {
    for (int i=0; i<replicas; i++) {
      String key = ch.getNodeHashKey(node, i);
      System.out.println(node.ip + ": " + key + "\t" + ch.getNodeHash(key));
    }
  }
  
  String[] requests = new String[1000];
  for (int i=0;i<requests.length; i++) {
    requests[i] = getRandomStr(16);
    System.out.println(ch.get(requests[i]));
  }
}
```
# 参考
 - [Consistent hashing](http://michaelnielsen.org/blog/consistent-hashing/)
 - [Consistent Hashing-Tom White](http://www.tom-e-white.com/2007/11/consistent-hashing.html)
 - [MemCache超详细解读](http://www.cnblogs.com/xrq730/p/4948707.html)
 - [Programmer’s Toolbox Part 3: Consistent Hashing](http://www.tomkleinpeter.com/2008/03/17/programmers-toolbox-part-3-consistent-hashing/)