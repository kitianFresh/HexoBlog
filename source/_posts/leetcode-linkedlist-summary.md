---
title:
  LeetCode 系列之链表
categories:
- DataStructure & Algorithm 
tags:
- LeetCode
- Data structure
- Algorithm
- LinkedList
---

# 链表操作总结

**链表作为线性存储结构中的一种（另一种是数组），使用非常广泛，他的删除和插入操作非常高效，不涉及节点的移动（相比数组而言），但是对于查找操作即使能拿到节点地址，平均也要O(n)的时间复杂度，而数组根据索引可以直接计算元素的位置，其查找为O(1)。**

**链表在空间组织上是离散在内存之中**，而**数组是连续存在于内存之中**，因此**链表对内存的利用率较高，而且不会很苛刻，只要有足够一个节点的空间，就可以增加元素**，但是**数组对内存的要求比较苛刻，内存紧张的时候数组就会由于找不到连续内存而无法继续存储。**

数组和链表都可以采用动态申请和静态申明的方式申请空间。你会惊讶链表也可以通过静态申明组织起来吗？
 - 数组的动态申请和静态申明：
   * C/C++
    ```C++
    // 动态申请（堆中）
    int *p = (*int)malloc(sizeof(int)*9);

    // 静态申明（栈中）
    int[] array = {1,2,3,4,5,6,7,8,9};
    ```
  * Java
   ```java
    // 动态申请（堆中）
    int[] array = new int[9];

    // 静态申明（栈中）
    int[] array = {1,2,3,4,5,6,7,8,9};
   ```

 - 链表的动态申请：
   * C/C++
 	```C++
 	ListNode * node = (ListNode*)malloc(sizeof(ListNode));
 	```
   * Java
   	```java
   	ListNode node = new ListNode();
   	```

链表的静态申明是一种比较有趣的数据结构，**链表和数组的组合**，开源代码中有这种写法！这更像是一个trick，不是什么新的数据结构，具体操作就是**申请一块静态数组，但是数组中的每一个元素是一个链表节点，然后链表节点显式链接在一起！**
```C++
typedef struct DataNode
{
    char*   cmd;
    char*   desc;
    int     (*handler)();
    struct DataNode *next;
} tDataNode;

int show_egg(const char *filename);
int cmd_help();
int cmd_quit();
int cmd_other();
int cmd_buddha();
int cmd_dogz();
int cmd_girl();
int cmd_moo();
int cmd_pet();
int cmd_rocket();

static tDataNode head[] = 
{
    {"help",    "this is help cmd!",            cmd_help,   &head[1]},
    {"version", "menu program v1.0",            NULL,       &head[2]},
    {"quit",    "quit the program",             cmd_quit,   &head[3]},
    {"buddha",  "print a buddha Easter egg",    cmd_buddha, &head[4]},
    {"dogz",    "print a dogz Easter egg",      cmd_dogz,   &head[5]},
    {"girl",    "print a girl Easter egg",      cmd_girl,   &head[6]},
    {"moo",     "print a moo Easter egg",       cmd_moo,    &head[7]},
    {"pet",     "print a pet Easter egg",       cmd_pet,    &head[8]},
    {"rocket",  "print a really bad Easter egg",cmd_rocket, NULL}
};
```
其实就是先声明一个链表结构的结构体，然后再以静态方式申明一个该结构体数组，数组元素的next成员都以字面形式链接到下一个元素，这样不就是数组+链表了么！你既可以按数组方式head[i]存取，也可以按照链表方式p->next存取；

直接静态的把这个链表建立起来！！重要的是你知道你的下一个节点的地址就行，静态情况下可以通过数组索引方式可以获得地址，然后就链接起来了！虽然这个数据会在编译之后就存在而非动态创建，会增大二进制文件数据的大小，但是对代码的可读性和可维护性来说值得，而且如果提前已经知道这块内存不论如何都是要占用的，这样做会更加高效，因为动态申请还会降低速度。

**静态和动态的唯一区别就是在于编译器和操作系统，静态其实就是编译器提前申请了内存，即在编译时期就确定了这些数据需要压栈，因为长度类型都是已知的，而动态就是编译时期无法确定大小，只有程序在执行的时候才会决定到底需不需要这块内存，此时由操作系统帮助完成堆内存的申请，产生系统调用的巨大开销！**

链表也分很多中，根据方向有单向链表，双向链表，根据收尾是否连接有循环链表，不循环链表，组合在一起，就有**单向不循环，单向循环，双向不循环，双向循环**四种。

以下只讨论 单向不循环链表。采用Java语言编写代码，由于Java具有GC，因此删除操作不用显式的释放内存，所以和C/C++写法会不一样，不会出现一个temp变量保存做free操作。

# 单向不循环链表
**单向不循环链表因其结构的特殊性**，对他进行操作有很多限制，比如**无法根据当前节点找到前驱，只能找到后继**。我把 链表的第一个元素节点称为 首节点， 而空头我把他称为 冗余头结点。
## 链表遍历查找
这是一个基础性问题，因为往往删除，修改，插入之前都是先涉及查找的，因此先来讲解查找问题。
 - [234. Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/?tab=Description)
 - [160. Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/?tab=Description)
 - [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/?tab=Description)
 - [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/?tab=Description)

### 链表找中点和倒数第K个点
我们很容易想到，遍历这个链表一次，求其总长度n，然后再从头开始遍历到你 n/2 的位置即可！但是如果不求长度呢？这里就要使用**经典的双指针的异步同速版本用于找到倒数第K个节点，也可以使用另一个版本是双指针的同步异速版本（用于找环儿)。**
```java
	// two pointers without length find mid of linkedlists, fast pointer is twice as fast as slow pointer
	public ListNode findMid(ListNode head) {
		if (head == null || head.next == null)
			return head;
		ListNode fast, slow;
		fast = slow = head;
		while (fast != null && fast.next != null && fast.next.next != null) {
			fast = fast.next.next;
			slow = slow.next;
		}
		return slow;
	}
	
	// two pointers without length find countdown kth Node, ealry pointer is early k step than later pointer
	public ListNode findCountdownKth(ListNode head, int k) {
		if (head == null || head.next == null)
			return head;
		ListNode early, later;
		early = later = head;
		int step = 1;
		while(step<k) {
			early = early.next;
			step ++;
		}
		while(early.next!=null) {
			early = early.next;
			later = later.next;
		}
		return later;
	}

```
### 链表找环问题系列


## 链表删除
 - [237. Delete Node in a Linked List](https://leetcode.com/problems/delete-node-in-a-linked-list/?tab=Description)
 - [203. Remove Linked List Elements](https://leetcode.com/problems/remove-linked-list-elements/?tab=Description)
 - [83. Remove Duplicates from Sorted List](https://leetcode.com/problems/remove-duplicates-from-sorted-list/?tab=Description)
 - [82. Remove Duplicates from Sorted List II](https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/?tab=Description)
 - [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/?tab=Description)

经典的链表删除问题，一个就是删除链表的一个节点至少需要多少个指针？要回答这个问题，就需要知道具体是什么类型的删除操作！
### 根据内容删除
**如果是根据节点的data内容删除，那么必须要使用至少两个指针，一个是当前指针，一个是前驱指针！**
```java
	public ListNode removeElements(ListNode head, int val) {
        if (head == null) return head;
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode p = dummy.next;
        ListNode pre = dummy;
        while (p!=null) {
            if (p.val == val) {
                pre.next = p.next;// java 显式不用释放
                p = pre.next;
            }
            else {
                pre = p;
                p = p.next;
            }
        }
        return dummy.next;
    }
```

### 根据节点地址删除

**如果是根据节点地址，直接删除该节点（尾节点除外），那么只需要一个当前欲删除节点的指针就够了**，就是覆盖的思想，**删除即覆盖，计算机的删除可以从覆盖的角度理解；时间复杂度是 O(n),太大了**

```java
	public void deleteNode(ListNode node) {
        if (node == null) return;
        ListNode p = node;
        while (p.next!=null) {
            p.val = p.next.val;
            if (p.next.next == null) { //判断是否复制到了最后一个元素，是则将其去掉即可，即前一个元素的next指针置空，跳出循环
                p.next = null;
                break;
            }
            p = p.next;
        }
    }

```

**直接复制当前欲删除节点的后一个到欲删除节点，然后删除后一个节点就行了，这个时候就知道他的前驱指针了！时间复杂度 O(1)**
```java
	public void deleteNode(ListNode node) {
        if (node == null) return;
        ListNode p = node;
        // 直接复制后一个节点data内容给当前欲删除的节点，然后就可以删除后一个节点了，时间复杂度 O(1)
        p.val = p.next.val;
        p.next = p.next.next;
    }
```
### 删除变种
然后就是删除的变种题目了，就是在基本的删除操作上加上一些条件等等，总结的一个技巧就是 冗余头结点的使用非常重要，这会让你的代码变得简洁优雅！**首节点可能改变或删除的，或者需要计数的，增加冗余头结点会简化代码**，以下是 82 和 83 有序链表的删除的应用；

首节点一定不会被删除，因此可以不使用冗余节点
```java
    public ListNode deleteDuplicates(ListNode head) {
        if (head == null || head.next == null) return head;
        ListNode pNode = head;
        while (pNode!=null && pNode.next!=null) {
            if (pNode.val == pNode.next.val) { // 相等删除后继
                pNode.next = pNode.next.next; // java不用释放内存 C/C++ temp = pNode->next; pNode->next = pNode->next->next; free(temp);
            }
            else {
                pNode = pNode.next;
            }
        }
        return head;
    }
```

首节点有可能被删除的，所以使用冗余节点更方便，另外此题相等往前一直跳很重要，可以跳过，就相当于删除了，其他写法很麻烦
```java
    public ListNode deleteDuplicates(ListNode head) {
        if (head == null || head.next == null) return head;
        ListNode dummy = new ListNode(-1); // 因为链表首节点有可能被删除，因此加入冗余节点，方便操作
        dummy.next = head;
        ListNode pre,p; // 因为是删除内容相等的，因此需要前驱指针
        pre = dummy;
        p = head;
        while (p!=null && p.next!=null) {
            if (p.val != p.next.val) {
                pre = p;
                p = p.next;
            }
            else {
                while (p!=null && p.next!=null && p.val == p.next.val) { // 相等一直往前走
                    p = p.next;
                }
                pre.next = p.next; // 跳过中间重复的元素
                p = p.next;
            }
        }
        return dummy.next;
    }
```
删除倒数第N个元素，难点就是链表不能倒着计数！因此这个题目首先就转换为查找链表的倒数第N-1个元素了！
代码见[NodeRemove.java](./NodeRemove.java)

## 链表逆序递归思想
 - [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/?tab=Description)
 - [92. Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii/?tab=Description)
 - [25. Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group/?tab=Description)

### 就地逆序
链经典的逆序就是链表的就地逆序，即直接在遍历原来的链表的过程中完成逆序，其实就是**使用链表头插法(天然逆序)重新建立链表**。由于链表首节点会发生变化，因此这里考虑dummy。

经典头插法就地逆序，一般会使用一个多余的空头，这样写起来更方便，不用判断是不是头结点，从而会导致代码做特殊处理，但是在Java中申请heap空间是耗费时间的
```java
    public ListNode reverseList(ListNode head) {
	    if (head == null || head.next == null) return head;
        ListNode dummby = new ListNode(0);
        dummby.next = null;
        ListNode pNode = head;
        ListNode pTemp = null;
        for (;pNode != null;) {
            pTemp = pNode;
            pNode = pNode.next;
            pTemp.next = dummby.next;
            dummby.next = pTemp;
        }
        return dummby.next;
	}
```
	
因此这个版本使用stack空间，不使用堆，直接申明引用变量；
	
```java
	public ListNode reverseList1(ListNode head) {
		if (head == null || head.next == null) return head;
        ListNode newHead = null;
        for (;head != null;) {
            ListNode next = head.next;
            head.next = newHead;
            newHead = head;
            head = next;
        }
        return newHead;
	}
```
	
就地逆序链表的递归版本，假设后面 n-1个元素已经逆序，那么只需要将首节点插入到逆序的n-1个元素的最后面就行了。1 -> 2 -> 3 -> 4 -- n-1 -> n -> @ ,假设我们已经对 2到n做好了逆序，那么 2 肯定在已经逆序的链表最后， 然后又因为 1.next 指向 2，在求解子问题过程中并没有改变，因此 1.next.next = 1即可将开头元素添加至结尾；
```java
	public ListNode reverseList2(ListNode head) {
		if (head == null || head.next == null) return head;
		ListNode pNode = reverseList2(head.next);
		head.next.next = head;
		head.next = null;
		return pNode;
	}
```
### 区间逆序
采用冗余头结点简化代码，首先遍历到第m个节点前驱，然后从这里开始使用头插法逆序n-m+1个元素，然后在接上后续的元素即可；

```java
	public ListNode reverseBetween(ListNode head, int m, int n) {
        if (head == null || head.next == null) return head;
        if (m == n) return head;
        //遇到头结点特殊处理或者需要计数的时候，采用冗余头结点会达到意想不到的效果；
        ListNode dummby = new ListNode(0);
        dummby.next = head;
        ListNode start = dummby;
        ListNode pre = dummby;
        ListNode end = dummby;
        //首先找到第m个节点的前驱，因为是单向链表；
        for (int i=1; i<m; i++) pre = pre.next;
        
        // 定位首位
        start = pre.next;
        end = pre.next;
        
        // 从pre节点开始做链表头插法实现逆序，当然要保存start的位置，到时候会变成末尾，然后直接接上end即可
        ListNode temp;
        pre.next = null;
        for (int i=1;i<=n-m+1;i++) {
        	temp = end;
        	end = end.next;
        	temp.next = pre.next;
        	pre.next = temp; 
        }
        start.next = end;
        return dummby.next;
    }
```
### 分组逆序
递归版本的实现；由于是每 K 个 为一组，因此 减少一组， 剩余部分是同样的子问题，假设后面 N- K 个已经 reverseKGroup了，则只需要前面 K 个元素 逆序， 然后连接上 后面 N-K 个 已经 reverseKGroup 的即可；
```java
	public ListNode reverseKGroup(ListNode head, int k) {
        if (head == null || head.next == null) return head;
        ListNode dumby = new ListNode(0);
        dumby.next = null;
        
        ListNode node = null;
        ListNode pNode = head;
        ListNode pTemp;
        int i;
        for (i=0;pNode!=null;) {
            pTemp = pNode;
            pNode = pNode.next;
            pTemp.next = dumby.next;
            dumby.next = pTemp;
            i ++;
            
            if (i % k == 0) {
                node = reverseKGroup(pNode, k);
                break;
            }
        }
        // 如果 i 是 小于 k 的， 说明这一部分不应该逆序，再逆序回去
        if (i < k) {
        	// pNode = head; // 严重错误的写法，此时的head 是以前 所指向的 head， 现在他已经在尾部了，现在的头部是 dumby.next;
        	pNode = dumby.next;
        	dumby.next = null;
            for (;pNode != null;) {
            	 pTemp = pNode;
                 pNode = pNode.next; // 先走，不然会被修改
                 pTemp.next = dumby.next;
                 dumby.next = pTemp;
            }
            return dumby.next;
        }
        // 尾巴指向 后面 reverseKGroup 返回的结果
        head.next = node; 
        return dumby.next;
    }
```
以上代码可以参考[ReverseLinkList.java](./ReverseLinkList.java)

## 链表重排,移位
 - [143. Reorder List](https://leetcode.com/problems/reorder-list/?tab=Description)
 - [328. Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list/?tab=Description)
 - [61. Rotate List](https://leetcode.com/problems/rotate-list/?tab=Description)
 
```java
 // 递归版本，当问题规模变大时，时间过长，而且可能会产生栈溢出
	public void reorderList(ListNode head) {
        if (head == null || head.next == null || head.next.next == null) return;
        ListNode pNode = head;
        ListNode preNode = head;
        for (;pNode.next != null;) {
            preNode = pNode;
            pNode = pNode.next;
        }
        preNode.next = null;
        ListNode newHead = head.next;
        reorderList(newHead);
        
        pNode.next = newHead;
        head.next = pNode;
    }
	
/**
* 1 2 3 4 5 6 7 8 9 10 
* 1 10 2 9 3 8 4 7 5 6 
*/
//非递归实现，仔细观察发现，重排序的链表其实是由 1 2 3 4 5 和 10 9 8 7 6 两个链表交叉合并的结果，因此只要找到中点位置，然后将后半部分逆序，然后再交叉合并即可！
```
以上问题的源代码见[ListRotate.java](./ListRotate.java) 和 [EvenOdd.java](./EvenOdd.java)

## 链表合并
 - [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/?tab=Description)
 - [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/?tab=Description)
 - [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers/?tab=Description)
 - [445. Add Two Numbers II](https://leetcode.com/problems/add-two-numbers-ii/)

### 合并两个有序链表
简洁紧凑的不用跳出循环后再判断的是否为空的代码，利用了算法导论中最大界哨兵的思想！
```java
	public ListNode mergeSortedLists(ListNode headA, ListNode headB) {
		if (headA == null || headB == null)
			return headA == null ? headB : headA;
		ListNode dummy = new ListNode(-1);
		dummy.next = null;
		ListNode tail = dummy;
		ListNode bound = new ListNode(Integer.MAX_VALUE); 
		// 利用 bound最大界，让代码更加compact & elegant;见算法导论
		while (headA != bound && headB != bound) {

			headA = (headA == null ? bound : headA);
			headB = (headB == null ? bound : headB);
			while (headA != null && headB != null && headA.val < headB.val) {
				tail.next = headA;
				tail = headA;
				headA = headA.next;
			}

			while (headA != null && headB != null && headA.val >= headB.val) {
				tail.next = headB;
				tail = headB;
				headB = headB.next;
			}
		}
		return dummy.next;
	}

```

### 合并K个有序链表--联想到归并排序，分治法
第一个版本是**递归版本**，即**基于归并排序算法的分治思想**，合并多个 已排序的链表，可以**将该问题分为两个子问题，即现将这些链表分为两部分 left 和 right，如果 left 和 right 已经合并完成并返回  l1 和 l2，那么最后将 l1 和 l2 进行两个有序链表的合并即可！**
```java
	public ListNode mergeKLists(ListNode[] lists) {
		if (lists == null) return null;
		if (lists.length == 1) return lists[0];
		if (lists.length == 2) {
			return mergeTwoLists(lists[0], lists[1]);
		}
		int len = lists.length;
		ListNode[] left = new ListNode[len/2];
		ListNode[] right = new ListNode[len/2 + len%2];
		int i,j;
		for (i=0; i<left.length; i++) {
			left[i] = lists[i];
		}
		for (j=0; j<right.length; j++) {
			right[j] = lists[j+left.length];
		}
		
		ListNode l1 = mergeKLists(left);
		ListNode l2 = mergeKLists(right);
		
		return mergeTwoLists(l1, l2);
        
    }
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if(l1 == null) return l2;
		if(l2 == null) return l1;
		if(l1.val < l2.val){
			l1.next = mergeTwoLists(l1.next, l2);
			return l1;
		} else{
			l2.next = mergeTwoLists(l1, l2.next);
			return l2;
		}
    }
```
由于每一次递归的每一层都需要复制数组，而递归层数是 logK，数组长度 K，空间复杂度是 O(K\*logK); 时间复杂度 O(nlgn)

以上版本是递归，如果写成非递归的形式，可以采用优先队列，即**每次从待排序链表中取出最小的节点，每取出来一个，就加入到合并后的链表中，然后将该节点所在链表移动一个位置，每次取K个链表头节点的最小值**,可以使用**优先队列**。优先队列的空间始终不大于 K，因为每放入一次就取出来一个，初始化时将 K 个链表的头结点分别放入即可；即空间复杂度是 O(K)， 而时间复杂度是 O(S\*logK) ; S 是链表的总元素个数；
```java
	public ListNode mergeKLists(ListNode[] lists) {
        if (lists.length == 0) return null;
        if (lists.length == 1) return lists[0];
        PriorityQueue<ListNode> pq = new PriorityQueue<ListNode>(lists.length,new Comparator<ListNode>(){
            @Override
            public int compare(ListNode o1,ListNode o2){
                if (o1.val<o2.val)
                    return -1;
                else if (o1.val==o2.val)
                    return 0;
                else 
                    return 1;
            }
        });
		
		ListNode dumby = new ListNode(0);
		dumby.next = null;
		ListNode tail = dumby;
		for (int i=0;i<lists.length;i++) {
		    if (lists[i] != null) {
			    pq.add(lists[i]);
		    }
		}
		
		ListNode node = null;
		while (!pq.isEmpty()) {
			node = pq.poll();
			tail.next = node;			
			if (node.next != null) {
				pq.add(node.next);
			}
			tail = node;
		}
		return dumby.next;
    }

```
### 链表版本的大数相加
简介紧凑的代码；
```java
	public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        if (l1 == null || l2 == null) return l1==null?l2:l1;
		ListNode dummby = new ListNode(0);
        dummby.next = null;
        ListNode tail = dummby;
        int carry = 0, sum =0;
        for (;l1!=null || l2!=null;) {
            sum = (l1==null?0:l1.val) + (l2==null?0:l2.val) + carry;
            carry = sum / 10;
            ListNode node = new ListNode(sum % 10);
            tail.next = node;
            tail = tail.next;
            l1 = (l1==null?null:l1.next);
            l2 = (l2==null?null:l2.next);
        }
        if (carry == 1) {
            tail.next = new ListNode(1);
        }
		return dummby.next;
    }
```

## 基于链表数据结构排序

### 插入排序
** 比较难写正确，虽然和数组的插入排序思想一样，由于不能从后往前比较，只能每次都从头开始比，很容易指错节点，为了防止从头比较的指针超过当前节点，采用提前判断的写法，即 p.next.val < pNode.val; 这样 p 就无法到达 pNode 了。 **
```java
	public ListNode insertionSortList(ListNode head) {
		if (head == null || head.next == null)
			return head;

		// 首节点可能会动，因此添加冗余节点
		ListNode dummy = new ListNode(-1);
		dummy.next = null;
		ListNode p, next, pNode;
		pNode = head;
		while (pNode != null) {
			next = pNode.next; // pNode 前进 不要放在最后
			p = dummy;
			// 采用 p.next 和 pNode 提前比较，就不会让 p 到达 pNode 了
			while (p.next != null && p.next.val < pNode.val) {
				p = p.next;
			}

			// 将pNode插入到 p 的前一个位置
			pNode.next = p.next;
			p.next = pNode;

			pNode = next;

		}
		return dummy.next;
	}
```
### 归并排序
实现 O(nlgn) 的复杂度，比较简单，思路和数组归并排序一样，找到中点，拆分成左右两个子问题，然后合并！
```java
	// O(nlgn) 给链表排序，归并或者快速排序，这里采用归并
	public ListNode sortList(ListNode head) {
		if (head == null || head.next == null)
			return head;
		ListNode pNode = findMid(head);
		ListNode left;
		ListNode right;

		right = sortList(pNode.next);
		pNode.next = null;
		left = sortList(head);
		return mergeSortedLists(left, right);
	}
```
## 其他
随机链表的深度拷贝问题
 - [138. Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer/?tab=Description)
 
# 小结
 1. **链表头插法是天然的逆序，尾插法是顺序的。头插法多用于逆序操作。**
 2. **多使用冗余头结点可以简化代码！一旦涉及到链表首节点可能修改，或者需要计数等操作时，使用冗余头结点可能更好写代码。**
 3. **链表删除操作分为根据内容删除和节点地址删除，前者至少需要两个指针(前驱指针和欲删除节点指针)，而后者只需要欲删除节点指针一个就行了。**
 4. **防止前驱指针的越界，多数情况下可以考虑 p.next.val 的提前判断。比如插入排序中。**
 5. **双指针法根据速度(fast & slow)和同异步(early & later)可以不求长度解决两大类问题，找环儿和找倒数第K个节点。**
 6. **链表操作也可以使用哨兵技术，冗余头结点是一种，添加冗余尾节点作为上界（bound）从而简化合并链表操作的代码，使其更加优雅紧凑。**
 7. **递归分治有时候也可以简化链表的操作。**
 8. **时刻注意操作过程中指针（Java中是引用）的指向是否改变。**

