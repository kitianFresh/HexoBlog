---
title:
  LeetCode 系列之排列与组合
date:
  2017-05-15
categories:
- DataStructure & Algorithm 
tags:
- LeetCode
- Data structure
- Algorithm
- Permutation
- Combination
- Shuffle
---

# 有趣的组合数学
组合数学中有很多有趣的问题，比如鸽巢原理、基本的全排列和组合问题、各式各样的取球问题、卡特兰数等，在计算机科学中应用非常广泛。

 - 比如 鸽巢原理 在分布式系统容错和一致性 保障的时候，往往采用的是多读多写的手段，写必须过半才算成功，读的时候读取一半以上就一定能够读到最新数据，根据数据版本号最新即可拿到。这里就用到了鸽巢原理。
 - 比如 数组混洗操作，一般采用的是全排练方法，即可得到所有的混洗。
 - 比如 卡特兰数，太多的树、栈和递归问题都是一种卡特兰数。例如，栈的所有合理出栈顺序，二叉树的不同构造方式等。

思考排列问题和组合问题的时候，往往采用递推和递归的手段是最容易理解的，而且程序也是最容易写的。但是不同的递归方式会导致不一样的程序和适应能力，有些递归可能会非常难以适应新题型的限制条件。

## 排列问题

设 多重集合 $ S=\\{ n\_1 \cdot a\_1, n\_2 \cdot a\_2, ... n\_k \cdot a\_k \\}, n = n\_1 + n\_2 + ... + n\_k$. 从中取出 r 个数做排列，这就是排列问题。

当 $r > n$ 时，排列总数有 $ N = 0 $;
当 $r = n$ 时，排列总数有 $ N = \frac{n!}{n\_{1}!n\_{2}! \cdot\cdot\cdot n\_{k}! }$;
当 $r < n$ 时，且有 $ \forall n\_i \geqslant r $, 则排列总数 $ N = k^r $;

### k个不同元素构成的全排列（元素不可以重复使用， 不放回的取出元素）

当 $r = n$ 且 $n\_1 = n\_2 = ... = n\_k = 1$ 时候，此时 $ r = k = n$ 就是我们俗称的全排列。

#### 1. 输出 n 位数字的全排列
我们知道公式是 $n!$ 其实就是假设 n-1 个元素的全排列已知为 p(n-1), 那么 n 个元素的全排列就是对 n-1 元素的每一个排列，都可以将第 n 个数插入到 n 个位置形成新的 n 长度的排列，因此 $p(n) = n p(n-1) = n!$.
因此，此题的首选方式就是递归求解。

```java
	// 法一 子问题划分, 自顶向下的递归法, 数学公式清晰. Permute[n] = [insertEveryWhere(elem) for elem in Permute[n-1]]
	  public List<List<Integer>> permuteI(int[] nums) {
        return permute(nums, nums.length - 1);
    }
    
    public List<List<Integer>> permute(int[] nums, int n) {
        List<List<Integer>> res = new ArrayList<List<Integer>>();
        if (n == -1) {
            res.add(new ArrayList<Integer>());
            return res;
        }
        if (n == 0) {
            List<Integer> e = new ArrayList<Integer>();
            e.add(nums[0]);
            res.add(e);
            return res;
        }
        List<List<Integer>> subs = permute(nums, n - 1);
        for (List<Integer> p : subs) {
            int size = p.size();
            for (int i = 0; i <= size; i ++) {
                ArrayList<Integer> subP = new ArrayList<Integer>(p);
                subP.add(i, nums[n]);
                res.add(subP);
            }
        }
        return res;
    }
```

另一种方法是采用DFS深度优先。

```java
    // 法三 DFS 采用深度优先搜索该问题的解空间树, 将解存在一个数组 path 中, 从第一个位置开始选择剩余可以选择的元素,直到选到最后一个元素;
    // 搜索返回的时候记住必须标记会被选择的元素为 还未选择,从新开始搜索树的下一个兄弟分支;
    public List<List<Integer>> permuteI2(int[] nums) {
        List<List<Integer>> res = new ArrayList<List<Integer>>();
        int n = nums.length;
        boolean[] leftedNums = new boolean[n];
        int[] path = new int[n];
        dfs(res, leftedNums, nums, path, n);
        return res;
    }
    
    public void dfs(List<List<Integer>> res, boolean[] leftedNums, int[] nums, int[] path, int n) {
        if (n <= 0) {
            ArrayList<Integer> t = new ArrayList<Integer>();
            for (int p: path) {
                t.add(p);
            }
            res.add(t);
            return;
        }
        int size = leftedNums.length;
        for (int i = 0; i < size; i ++) {
            if (leftedNums[i]) continue;
            int selected = nums[i];
            path[n-1] = selected;
            leftedNums[i] = true;
            dfs(res, leftedNums, nums, path, n - 1);
            leftedNums[i] = false;
        }
    }
```
但是上述两种方式，在面临有重复元素的时候，会非常麻烦。因为剔除重复解的过程很麻烦。因此，我们还有第三种方法。我们在后面的题目中讲解。

### k个不同元素的全排列（元素可以重复使用，有放回的取出元素）
当 $ r < n $ 时，且有 $ \forall n\_i \geqslant r $, 则排列总数 $N = k^r$. 此时 $r = k$ 有 $N = k^k$.

使用该集合选出 k 个组合成一个序列，共有多少种。

#### 2. 输出所有的 n 位二进制数
$S=\\{ \infty \cdot 0, \infty \cdot 1 \\} $. 此时属于  $ r < n$ 且有 $\forall n\_i >= r$, 则排列总数 $N = k^r$
题目的意思就是有两个不同的数 0 和 1， 由这两种元素组成的 n 位数的全排列。思路也比较固定，还是和上面一样的自顶向下的递推或者是自底向上的DFS方式。

```java
 // DFS T(n) = 2 * T(n-1) + O(1); NP 2^n
 	public void nBits(int n, int[] A) {
 		if (n <= 0) {
 			System.out.println(Arrays.toString(A));
 		}
 		else {
 			A[n-1] = 0;
 			nBits(n-1, A);
 			A[n-1] = 1;
 			nBits(n-1, A);
 		}
 	}
```

#### 3. 输出由字母 "aabbccc" 构成的所有的全排列。这里就会出现重复选取的问题
$S=\\{ 2 \cdot a, 2 \cdot b, 3 \cdot c \\}$， 此时属于 $r = n$ 时，排列总数有 $ N = \frac{n!}{n\_{1}!n\_{2}! \cdot\cdot\cdot n\_{k}!}$；

对于这种情况，我们就来采用第三种方式，这是另外一种思维方式，想法是把**字符串分成左右两部分，第一部分是第一个字符，第二部分是后面的字符串．第一个字符可以有多种不同的选择，从第一个到最后一个(如果没有重复元素的话), 选定一个之后，第二部分就可以作为相同性质的子问题。**

这里是对全排列的另一种演算方式，**全排列的互斥完备集合由第一个位置元素来划分得到，只要首个元素不一样，就是不同的解，所有的解构成全部解空间。**

交换任意两个位置元素的过程，由基础排列开始，第一个元素可以和第二部分任意元素交换，交换之后子问题一样的方式进行, 但是**子问题求完之后再换回来就可以恢复原样。**

这种方法的好处是有的，就是当**含有重复元素的时候，(可以对字符串排序)，第一个元素和第二部分元素交换的时候，如果发现是重复，就不必交换了。**

```java
  public ArrayList<String> Permutation(String str) {
    char[] chs = str.toCharArray();
    //Arrays.sort(chs);
    StringBuilder sb = new StringBuilder(String.valueOf(chs));
    ArrayList<String> res = new ArrayList<String>();
    if (str == null || str.length() == 0) return res;
    permute(res, sb, 0);
    return res;
  }
  
  public void permute(ArrayList<String> res, StringBuilder sb, int pos) {
    if (pos == sb.length()) {
        res.add(sb.toString());
        return;
    }
    for (int i = pos; i < sb.length(); i++) {
      if (i != pos && sb.charAt(i) == sb.charAt(pos)) continue; // 在不同的位置上相同的值,交换了也没用
        char temp = sb.charAt(i);
        sb.setCharAt(i, sb.charAt(pos));
        sb.setCharAt(pos, temp);
        permute(res, sb, pos + 1);
        temp = sb.charAt(i);
        sb.setCharAt(i, sb.charAt(pos));
        sb.setCharAt(pos, temp);
    }
    return;
  }
```

## 组合问题

### 全组合
#### 5. [Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/description/)
此题还是课使用自顶向下的递推方式，假设已知 n-1 长度的所有组合集合， 那么长度为 n 的组合集合就是在原来每个解的基础上加上新元素即可。

```java
	// 法一 自顶向下的递推方式
	  String[] digitsMap = new String[]{"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
    public List<String> letterCombinations(String digits) {
        return letterCombinations(digits, digits.length() - 1);
    }
    
    public List<String> letterCombinations(String digits, int n) {
        List<String> combs = new ArrayList<String>();
        if (digits == null || digits.length() == 0) return combs;
        String letters = digitsMap[digits.charAt(n) - '0'];
        if (n == 0) {
            
            for (int i = 0; i < letters.length(); i ++) {
                combs.add(String.valueOf(letters.charAt(i)));
            }
            return combs;
        }
        List<String> subCombs = letterCombinations(digits, n - 1);
        for (String subComb: subCombs) {
            for (int i = 0; i < letters.length(); i++) {
                combs.add(subComb + String.valueOf(letters.charAt(i)));
            }
        }
        return combs;
    }
    

```

第二种方式是采用 BFS 的方法，构建整个集合其实类似于构建一颗树的过程，从根开始，第一层选择第一个数字对应的所有字母，一次类推，得到所有的可能。

```java
    // 法二 采用queue ,类似于 bfs. 你可以把这个带限制的排列问题, 想象成一个树的构建过程, 如从根节点 第一个数字开始, 映射出多个字母孩子节点,
    // 接下来的第二个数字,对于上一层次的每一个字母,都可以再次映射出该层数字对应的字母组合,这样你可以使用queue保存上一层所有的结果, 然后从队列中取出
    // 上一层所有的结果,尾部添加新元素构成新的叶子节点.
	public List<String> letterCombinations1(String digits) {
        String[] digitsMap = new String[]{"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
        LinkedList<String> queue = new LinkedList<String>();
        if (digits == null || digits.length() == 0) return queue;
        queue.offer("");
        for (int i = 0; i < digits.length(); i ++) {
            while (queue.peek().length() == i) {
                String letters = digitsMap[digits.charAt(i) - '0'];
                String parent = queue.poll();
                for (char letter : letters.toCharArray()) {
                    queue.offer(parent + String.valueOf(letter));
                }
            }
        }
        return queue;
    }
```

#### 6. [Subsets](https://leetcode.com/problems/subsets/description/)
不带重复元素的子集。

```java
	// 解法一  自顶向下的递归
	public List<List<Integer>> subsets(int[] nums) {
        return subsets(nums, nums.length-1);
    }
    
    public List<List<Integer>> subsets(int[] nums, int end) {
        List<List<Integer>> res = new ArrayList<List<Integer>>();
        if (end < 0) {
            res.add(new ArrayList<Integer>());
            return res;
        }
        List<List<Integer>> res1 = subsets(nums, end - 1);
        List<List<Integer>> res2 = new ArrayList<List<Integer>>();
        for (List<Integer> l : res1) {
            ArrayList<Integer> b = new ArrayList<Integer>(l);
            b.add(nums[end]);
            res2.add(b);
        }
        for (List<Integer> l : res1) {
            res.add(l);
        }
        for (List<Integer> l : res2) {
            res.add(l);
        }
        
        return res;
    }
```
非递归方法

```java
       List<List<Integer>> res = new ArrayList<List<Integer>>();
       if (end < 0) {
           res.add(new ArrayList<Integer>());
           return res;
       }
       List<List<Integer>> res1 = subsets(nums, end - 1);
       res.addAll(res1);
       int lastSize = res1.size();
       for (int i = 0; i < lastSize; i++) {
           List<Integer> e = new ArrayList<Integer>(res.get(i));
           e.add(nums[end]);
           res.add(e);
       }
       
       return res;
```

#### 7. [Subsets II contains duplicated element](https://leetcode.com/problems/subsets-ii/description/)
对于含有重复元素的，这个时候添加新元素的时候，如果前面已经添加过一次，不能从头开始,因为从头到上一次构建开始这些都已经由上一个元素构建完了,不需要再构建了;因此直接使用该重复元素新构建的子集进行构建新子集;

```java
    public List<List<Integer>> subsetsWithDup(int[] nums) {
        Arrays.sort(nums);
        List<List<Integer>> res = new ArrayList<List<Integer>>();
        res.add(new ArrayList<Integer>()); // 空集合
        int begin = 0;
        for (int i = 0; i < nums.length; i ++) {
            if (i == 0 || nums[i] != nums[i-1]) begin = 0; // 当元素不相等的时候,从头开始取出所有的子集,再加上新元素,添加进来,和没有重复元素的时候一样
            // 当元素相等的时候,不能从头开始,因为从头到上一次构建开始这些都已经由上一个元素构建完了,不需要再构建了;因此直接使用该重复元素新构建的子集进行构建新子集;
            int size = res.size(); // 当前问题规模解的个数
            for (int j = begin; j < size; j ++) {
                ArrayList<Integer> sub = new ArrayList<Integer>(res.get(j));
                sub.add(nums[i]);
                res.add(sub);
            }
            begin = size;
        }
        return res;
    }
```