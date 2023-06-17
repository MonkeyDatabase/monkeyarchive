---
layout: post
title: 回溯法与剪枝
excerpt: "在刷算法题的过程中，有些复杂问题使用for循环暴力破解都无法解决，回溯法可有效解决这些复杂问题(组合、子集、切割、排列、棋盘)，因此本文记录一些回溯法相关的知识"
date:   2021-01-01 16:28:00
categories: [code]
comments: true
---

## 定义

1. 二叉树在递归的过程中都会有回溯的操作，只不过有些情况下利用了回溯实现功能，有些情况下没有利用回溯
2. 回溯函数实际上指的是递归函数
3. 回溯法是纯暴力搜索法。虽然它是暴力搜索，但是有一些问题没有办法写出使用for循环暴力搜的代码，所以要依靠回溯法将所有结果搜出来。
   * 组合
   * 切割
   * 子集
   * 排列
   * 棋盘
4. 回溯法的输入大多为一个数组.
5. 回溯法可以抽象为一个n叉树
   * 树的宽度由输入数组的长度决定
   * 树的深度由递归深度决定，因为递归一定是有终止的，终止后一层一层的向上 返回
6. 重要变量
   * 递归函数的参数
   * 递归函数的返回值，一般为void
   * 递归函数的终止条件
   * 单层递归逻辑
7. 回溯法实际上是收集路径的过程，从树的根节点到所有叶子节点上的路径的集合
   * 一维数组path，用于存放根节点到某个叶子节点的路径
   * 二维数组result，用于存放所有的path

## 模板

```java
public void backtracking(T[] nums,int start,.......){
    if(stopcondition){
        result.add(new ArrayList<T>(temp));
        return;
    }
    for(T t :T[]){
        //some code deal with t;
        backtracking(..........);
        //some code back deal t;
    }
}
```

## 例题

### 1、子集

题目地址为[Leetcode 78](https://leetcode-cn.com/problems/subsets/)

#### 1.1 解题思路

1. 在顺序无关的回溯中画出的递归树中，选择列表的元素都是全集中在选择路径后面的元素，用来实现顺序无关，因此状态变量应为选择列表的起始位置startindex，即标识每一层的状态
   * 全集为[1,2,3]
   * 已经选中了0号元素数值为1，即选择路径为[1]，则此时start为1(数组序号从0开始)时，选择列表为[2,3]，即只要把[2,3]的所有子集算出来拼到已选出的元素后就是选中0号元素的全部子集
2. 子集问题是收集树中**根节点到所有节点**的信息。

#### 1.2 程序代码

```java
class Solution {
    List<Integer> path=new ArrayList<Integer>();
    List<List<Integer>> result=new ArrayList<List<Integer>>();

    public List<List<Integer>> subsets(int[] nums) {
        backtracing(nums,0);
        return result;
    }

    public void backtracing(int[] nums,int startindex){
        
        result.add(new ArrayList<Integer>(path));

        for(int i=startindex;i<nums.length;i++){
            path.add(nums[i]);
            backtracing(nums,i+1);
            path.remove(path.size()-1);
        }
    }
}
```

> 本题要注意的点是每次向result集合中add的时候需要根据temp创建一个新的List\<Integer\>对象，否则result中存储的所有的都是指向temp对象的索引，经过一系列add()和remove()操作改动的都是这个对象，导致最终输出时只能一堆空集

### 2、子集Ⅱ

题目地址为[Leetcode 90](https://leetcode-cn.com/problems/subsets-ii/)

#### 2.1 解题思路

1. 子集为顺序无关的，然而本题的输入中含有重复的元素，所以需要先进行按大小排序以便于判断是否要处理该节点。
2. 每个节点处理逻辑
   * 如果这是第一个待选，则直接考虑处理
   * 如果不是第一个待选，则它可能与它相邻的前一个是同样的数值，而子集是顺序无关的，因此只有这个值与它相邻的前一个不同时才进行处理

#### 2.2 程序代码

```java
class Solution {
    List<Integer> path=new ArrayList<Integer>();
    List<List<Integer>> result=new ArrayList<List<Integer>>();

    public List<List<Integer>> subsetsWithDup(int[] nums) {
        Arrays.sort(nums);
        backtracing(nums,0);
        return result;
    }

    public void backtracing(int[] nums,int startindex){
        result.add(new ArrayList<Integer>(path));

        for(int i=startindex;i<nums.length;i++){
            if(i==startindex||nums[i]!=nums[i-1]){
                path.add(nums[i]);
                backtracing(nums,i+1);
                path.remove(path.size()-1);
            }
        }
    }
}
```

### 3、组合

题目地址为[Leetcode 77](https://leetcode-cn.com/problems/combinations/)

#### 3.1 解题思路

1. 题目要求求出1...n之间所有k个数的组合
   * 当k=2时，我们可以用两层for循环求出结果，然而目前组合所用的k是动态改变的，所以传统暴力搜索法无法使用，只能依靠于回溯法解决无法动态增加for循环的问题。
   * 回溯法就是用递归来实现动态多层for循环，深一层递归就是子一层for循环。
2. 组合问题是收集满足条件的**根节点到叶子节点**的信息。
3. 抽象一个树形结构，假设输入为n=4，k=2
   * [1,2,3,4]
     * 取1，[2,3,4]
       * 取2，[3,4]剩余，得到集合[1,2]
       * 取3，[2,4]剩余，得到集合[1,3]
       * 取4，[2,3]剩余，得到集合[1,4]
     * 取2，[3,4]
       * 取3，[4]剩余，得到集合[2,3]
       * 取4，[3]剩余，得到集合[2,4]
     * 取3，[4]
       * 取4，[]剩余，得到集合[3,4]
     * 取4，\[\]选择列表为空，得不到结果
   * 本题为求组合，组合是顺序无关的，所以其内部元素不可重复使用，因此，第一次取2后的选择列表中没有1，因为这种情况已经被包括在第一次取1的情况中，因此需要设置一个startindex来指定每次搜索的起始位置。
4. 终止条件：路径的长度达到了k，达到了收集条件
5. 单层搜索逻辑：每一个节点是一个for循环，遍历从startindex开始到最后的数组的元素
   * path收集当前遍历到的元素
   * 递归调用遍历剩余元素
   * 回溯，将当前遍历的元素从path中剔除

#### 3.2 程序代码

```java
class Solution {
    List<Integer> path=new ArrayList<Integer>();
    List<List<Integer>> result=new ArrayList<List<Integer>>();
    public List<List<Integer>> combine(int n, int k) {
        backtracing(n,k,1);
        return result;
    }

    public void backtracing(int n,int k,int startindex){
        if(k==path.size()){
            result.add(new ArrayList<Integer>(path));
            return;
        }
        for(int i=startindex;i<=n;i++){
            path.add(i);
            backtracing(n,k,i+1);
            path.remove(path.size()-1);
        }
    }
}
```

### 4、全排列

题目地址为[Leetcode 46](https://leetcode-cn.com/problems/permutations/)

#### 4.1 思路分析

1. 本题所给出的数字的序列中**不含重复**的数字，所以不需要剪枝
2. 组合的特点是收集路径，排列的特点是交换元素收集节点
3. 递归的结束条件是收集到的path长度等于输入序列元素的个数，
4. 与组合和子集**不同**的是排列是要收集所有输入序列的元素，即当startindex=10的时候，也是要从0开始遍历，而不是不要startindex前面的元素，此时需要遍历整个数组，而且已经选过的元素不能再次选，本题是**不含重复**的数字，所以只需要把**选过的值**缓存到一个Set集合中，每层逻辑中遍历整个数组而且选那些不在Set中的元素进行收集和递归。

#### 4.2 程序代码

```java
class Solution {
    List<Integer> path=new ArrayList<Integer>();
    Set<Integer> set=new HashSet<Integer>();
    List<List<Integer>> result=new ArrayList<List<Integer>>();
    public List<List<Integer>> permute(int[] nums) {
        backtracing(nums);
        return result;
    }

    public void backtracing(int[] nums){
        if(path.size()==nums.length){
            result.add(new ArrayList<Integer>(path));
            return;
        }
        for(int i=0;i<nums.length;i++){
            if(set.contains(nums[i]))
                continue;
            path.add(nums[i]);
            set.add(nums[i]);
            backtracing(nums);
            set.remove(nums[i]);
            path.remove(path.size()-1);
        }
    }
}
```

### 5、全排列II

题目地址为[Leetcode 47](https://leetcode-cn.com/problems/permutations-ii/)

#### 5.1 解题思路

1. 本题包含**重复**数字，所以4中只用一个Set标识值选没选过是不可以的
2. 例如[1,2,2,3]
   1. [1,2(1),3]和[1,2(2),3]是一样的，剪枝掉相同元素在同一层中的选择
   2. [1,2(1),2(2)]和[1,2(2),2(1)]是一样的，剪枝掉相同元素，在两个不同层位置颠倒的选择
3. 难点在于解决第二种剪枝操作
4. 首先要将这两个不同层的深度差距固定下来，即对数组进行排序，这样选[1,2(1)]紧接着后2(2)还没访问过就可以放入集合，当选[1,2(2)]时，再选2(1)的话，2(1)在数组中紧挨着2(2)，因为已经选了2(2),2(2)与2(1)相同，这种情况已经被[1,2(1)]后选2(2)包括。

#### 5.2 程序代码

``` java
class Solution {
    List<Integer> path=new ArrayList<Integer>();
    List<List<Integer>> result =new ArrayList<List<Integer>>();
    public List<List<Integer>> permuteUnique(int[] nums) {
        boolean[] visited = new boolean[nums.length];
        Arrays.sort(nums);
        backtracing(nums,visited);
        return result;
    }

    public void backtracing(int[] nums,boolean[] visited){
        if(path.size()==nums.length){
            result.add(new ArrayList<Integer>(path));
            return;
        }
        
        for(int i=0;i<nums.length;i++){
            if(visited[i]||i!=0&&nums[i]==nums[i-1]&&visited[i-1]==true)
                continue;
            path.add(nums[i]);
            visited[i]=true;
            backtracing(nums,visited);
            path.remove(path.size()-1);
            visited[i]=false;
        }
    }
}
```

### 6、分割回文串

题目地址为[Leetcode 131](https://leetcode-cn.com/problems/palindrome-partitioning/)

#### 6.1 解题思路

1. 切割问题是将原序列分割，与组合问题恰恰相反
2. 每一层递归加一条切割线，切割线的左右两侧均为回文串

#### 6.2 程序代码

```java
class Solution {
    List<String> path = new ArrayList<String>();
    List<List<String>> result = new ArrayList<List<String>>();
    public List<List<String>> partition(String s) {
        backtracing(s,0);
        return result;
    }

    public void backtracing(String s,int startindex){
        if(startindex==s.length()){
            result.add(new ArrayList<String>(path));
            return;
        }        
        for(int i=startindex+1;i<=s.length();i++){
            String str=s.substring(startindex,i);
            if(isPalindrome(str)){
                path.add(str);
            }else{
                continue;
            }
            backtracing(s,i);
            path.remove(path.size()-1);
        }
    }

    public boolean isPalindrome(String s){
        for(int i=0,j=s.length()-1;i<j;i++,j--){
            if(s.charAt(i)!=s.charAt(j))
                return false;
        }
        return true;
    }
}
```



## 总结

1. 重复：如果输入含有重复元素，需要对其进行排序，以便每次递归时辨别重复情况，因为此时重复元素是相邻的，使用数组下标就可以判断。
2. 子集
   * 顺序无关
   * startindex去重
   * 收集到达所有节点的路径
3. 组合
   * 顺序无关
   * startindex去重
   * 收集到达满足组合条件的叶子节点的路径
4. 排列
   * 顺序有关
   * visited去重
   * 收集所有长度与输入数据长度相同的路径
5. 分割
   * 分割点顺序无关
   * startindex去重
   * 收集到达叶子节点的路径