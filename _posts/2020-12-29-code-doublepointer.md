---
layout: post
title: 双指针法
excerpt: "在刷算法题的过程中，常常遇到双指针法，本文将记录一下双指针法的学习笔记"
date:   2020-12-29 13:20:00
categories: [code]
comments: true
---

## 定义

1. 双指针法也叫快慢指针
2. 双指针的实现
   * 在数组存储结构中是指两个代表下标的整数
   * 在链式存储结构中是指两个代表地址的指针
3. 时间复杂度：一般能在O(n)的时间解决问题
4. 两个指针的位置
   * 第一个元素和第二个元素
   * 第一个元素和最后一个元素

## 例题

### 1、汇总区间

题目地址为[Leetcode 228](https://leetcode-cn.com/problems/summary-ranges/)

#### 1.1 解题思路

1. 对于包含连续元素的一段区间，如果相邻元素的差值大于1，那么这两个元素一定不在一个区间
2. 题目中所给的数组是有序的，而且没有重复元素，此时两个相邻的元素要么等于1要么大于1
   * 差值等于1的放在同一段区间
   * 差值大于1的放在不同的区间
3. 根据输出，我们还需要知道一段区间的起始坐标，因此需要保存两个坐标，分别代表一段区间的两个分界点
4. 遍历到一个新元素，检查是否可以拓展原区间
   * 如果可以，则继续遍历
   * 如果不可以，则输出分区，并将该元素作为新区间的起始点
5. 尤其要注意最后一段区间也要放进结果集里面

#### 1.2 程序代码

```java
class Solution {
    public List<String> summaryRanges(int[] nums) {
        List<String> list=new ArrayList<>();

        for(int i=0,j=0;j<nums.length;j++){
            i=j;

            while(j+1<nums.length && nums[j+1]==nums[j]+1)
                j++;
            
            if(i==j)
                list.add(nums[i]+"");
            else
                list.add(nums[i]+"->"+nums[j]);
        }
        return list;
    }
}
```

### 2、盛最多水的容器

题目地址为[Leetcode 11](https://leetcode-cn.com/problems/container-with-most-water/)

#### 2.1 解题思路

1. 其实就是求最大矩形面积，宽为两个点x轴的距离，高为两个点中较矮的那个点的y值
2. 为了找盛水最多的容器，需要两点在x轴距离尽可能大的同时较矮的点要尽可能的高，所以使用双指针法时的两个指针分别位于第一个元素和最后一个元素
3. 题目中给出的示例为[1,8,6,2,5,4,8,3,7]
   * 初始时，左右指针分别指向两端，min(1,7)*8=8
   * 此时需要移动一个指针！那么移动哪一个呢？
   * 水量=两个指针中数字较小值*指针之间的距离
   * 如果移动数字较大的那个指针，那么不仅第一项不会增加，第二项还会减少1，导致水量一定减少
   * 所以！此时要移动数字较小的那个指针，此时第一项不一定增加，第二项减少1，所以此时水量可能增加，也可能减少，所以此时需要和原水量进行比较
   * 不断迭代，直至左右指针相遇

#### 2.2 程序代码

```java
class Solution {
    public int maxArea(int[] height) {
        int result=0;
        for(int i=0,j=height.length-1;i!=j;){           
            int temp=Math.min(height[i],height[j])*(j-i);
            result=Math.max(temp,result);
            if(height[i]>height[j])
                j--;
            else
                i++;
        }
        return result;
    }
}
```

### 3、平方数之和

题目地址为[Leetcode 633](https://leetcode-cn.com/problems/sum-of-square-numbers/)

#### 3.1 解题思路

1. 给定c，找满足a²+b²=c的a和b，如果用两个for循环遍历从0到c，那么当c为100000时，总共就要循环100000000000次核心语句，导致超时
2. 因为a²和b²，所以找两者的最大范围，即当一方为0时，另一方最大为√c，这样就大幅减少了遍历的次数
   * left指针指向0
   * right指针指向Math.sqrt(c)
   * 两个指针向中间移动，直至相遇，此时最多运行√c次核心语句
3. 接下来，需要确定每次移动那个指针，由于这里是遍历自然数，自然是有序的
   * 当left\*left+right\*right>c时，说明大了需要减少，则把right--
   * 当left\*left+right\*right<c时，说明小了需要增大，则把left--

#### 3.2 程序代码

```java
class Solution {
    public boolean judgeSquareSum(int c) {
        int temp=0,right=(int)Math.floor(Math.sqrt(c));
        for(int left=0;left<=right;){
            temp=left*left+right*right;
            if(temp>c)
                right--;
            else if(temp<c)
                left++;
            else
                return true;
        }
        return false;
    }
}
```

### 4、反转字符串中的元音字母

题目地址为[Leetcode345](https://leetcode-cn.com/problems/reverse-vowels-of-a-string/)

#### 4.1 解题思路

1. 分别使两个指针指向字符串的第一个字符和最后一个字符
2. 左指针向后寻找直到找到一个元音字母
3. 右指针向前寻找直到找到一个元音字母
4. 将两者进行交换，将左右指针各自向之间移动一
5. 不断迭代，直到左右指针相遇

#### 4.2 程序代码

```java
class Solution {
    public String reverseVowels(String s) {
        StringBuffer sb=new StringBuffer(s);
        char temp;
        for(int i=0,j=s.length()-1;i<j;){
            while(i<s.length()&&!isVowel(s.charAt(i)))
                i++;
            while(j>0&&!isVowel(s.charAt(j)))
                j--;
            if(i<j){
                temp=s.charAt(i);
                sb.setCharAt(i,s.charAt(j));
                sb.setCharAt(j,temp);
            }
            i++;
            j--;
        }
        return sb.toString();
    }

    public boolean isVowel(char c){
        if(c=='a'||c=='A'||c=='e'||c=='E'||c=='i'||c=='I'||c=='o'||c=='O'||c=='u'||c=='U')
            return true;
        else
            return false;
    }
}
```

> 两个while循环里将i和j的大小范围写在`&&`的左边，如果把它写在`&&`右边这个程序会报错：
>
> 1. 逻辑表达式是从左往右算
> 2. 一旦&&的左侧为假，右侧不必计算，结果一定是假
> 3. 一旦\|\|的左侧为真，右侧不必计算，结果一定为真
>
> 这样就能保证`&&`右侧的charAt()函数不报outOfRange的错误

### 5、环形链表

题目地址为[Leetcode 141](https://leetcode-cn.com/problems/linked-list-cycle/)

#### 5.1 解题思路

1. 两个指针

   * slow指针指向第一个元素
   * fast指针指向第二个元素

2. 移动速度

   * slow指针每次向前移动一个元素位置

   * fast指针每次向前移动两个元素位置

   * 每次移动，fast比slow多走一步

   * 当slow到达换链表尾连接到链表中的位置的node时，假设该node的序号为N(序号从0开始计数)，则fast已经在环中走了N+1步

   * 当fast追赶上slow时，fast比slow多走了一圈，此时fast==slow，即可认为有环

   * 其他计算

     * 当第一次相遇后，继续走K步再次相遇，说明环长为K。

     * 当第一次相遇时，slow还没走完链表，fast已经在环中循环了n圈，假设slow走了s步，fast走了2s步。假设链表长L，环入口与相遇点距离为x，链表头到环入口的距离为a，环长为r，

       ```txt
       ∵2s=s+nr
       ∴s=nr
       ∵s=a+x
       ∴a+x=nr
       ∴a+x=(n-1)r+r=(n-1)r+(L-a)
       ∴a=(n-1)r+(L-a-x)
       ```

       L-a-x为相遇点到环入口的距离，所以链表头到环入口的距离等于(n-1)循环内环长度加上相遇点到换入口点的距离。因此将fast指针重新指向表头，步速和slow指针一样每次移动一个元素的距离，当两指针相遇时，即为fast第一次到达环入口点

#### 5.2 程序代码

```java
public class Solution {
    public boolean hasCycle(ListNode head) {
        if(head==null||head.next==null)
            return false;
        ListNode slow=head,fast=head.next;
        while(fast!=slow){
            if(fast.next==null||fast.next.next==null)
                return false;
            slow=slow.next;
            fast=fast.next.next;
        }
        return true;
    }
}
```

### 6、通过删除字母匹配到字典里最长单词

题目地址为[Leetcode 524](https://leetcode-cn.com/problems/longest-word-in-dictionary-through-deleting/)

#### 6.1 解题思路

1. 题目要求找尽可能长的单词，如果长度相同，则在字典中位置靠前的单词，所以先对字典进行排序，使用静态内部类声明一个实现了Comparator接口的对象，其中实现compare方法，返回条件就是长度长的在前面，长度一样长则原来的顺序靠前就靠前
2. 双指针
   * 一个指针指向要删去内容的字符串的第一个字符
   * 一个指针指向字典中一个字符串的第一个字符
   * 依次遍历要删去内容字符串的字符，如果与字典中字符串相同，则第二个指针向后移动一个元素
   * 当第一个字符串遍历完成后，检查第二个指针是否已经指向第二个字符串的最后一个字符，即是否已经遍历完，即是否第一个字符串删掉某些字符后能成为第二个字符串

#### 6.2 程序代码

```java
class Solution {
    public String findLongestWord(String s, List<String> d) {
        Collections.sort(d,new Comparator<String>(){
            @Override
            public int compare(String s1,String s2){
                return s2.length()!=s1.length()? s2.length()-s1.length():s1.compareTo(s2);
            }
        }); 
        for(String ds:d){
            if(isSubSequence(s,ds))
                return ds;
        }
        return "";
    }

    public boolean isSubSequence(String s,String ds){
        int i=0,j=0;
        for(;i<s.length()&&j<ds.length();i++){
            if(s.charAt(i)==ds.charAt(j))
                j++;
        }
        return j==ds.length();
    }
}
```

### 7、无重复字符的最长子串

题目地址为[Leetcode 3](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

#### 7.1 解题思路

1. 要在一个字符串中找无重复字符的字符串，需要用双指针分别指向目标字符串的头和尾
2. 因此起始条件为
   * 左指针指向第一个元素
   * 右指针指向第二个元素
3. 由于要判断right正指向的元素在不在left到right中间，所以采用HashMap进行存储出现过的字符最后一次出现的位置，HashMap的特性是当put的时候如果之前有过相同的key则直接覆盖value
4. 当出现相同的时候，需要把left移动到重复字母在left和right之间最后一次出现的位置之后
5. 由于HashMap中存储的是它最后一次出现的位置，所以当冲突的时候，也可能冲突字符在left之前，因为每次left跳变都会跳到上一次冲突之后的位置，中间跳过了许多元素

#### 7.2 程序代码

```java
class Solution {
    public int result=0;
    public int left=0;
    public HashMap<Character,Integer> map=new  HashMap<Character,Integer>();
    public int lengthOfLongestSubstring(String s) {
        if(s.length()==0)
            return 0;
        map.put(s.charAt(0),0);
        for(int right=1;right<s.length();right++){
            if(map.containsKey(s.charAt(right))){
                left=Math.max(left,map.get(s.charAt(right))+1);
            }
            map.put(s.charAt(right),right);
            result=Math.max(result,right-left+1);
        }
        return result;
    }
}
```

### 8、奇偶链表

题目地址为[Leetcode 328](https://leetcode-cn.com/problems/odd-even-linked-list/)

#### 8.1 解题思路

1. 双指针o，t
   * o指向奇数节点链头
   * t指向偶数节点链头
2. 双指针otail，ttail
   * otail指向奇数链链尾
   * ttail指向偶数链链尾
3. 通过一个循环进行遍历，循换持续条件为ttail.next不为空，即有下一个奇数节点
   * 将下一个奇数节点连入otail后面
     * otail.next=ttail.next;
     * otail=otail.next;
   * 检查是否有下一个偶数节点otial.next不为空
     * 如果不为空，说明有下一个偶数节点
       * ttial.next=otail.next;
       * ttail=ttail.next;
     * 如果为空，说明没有下一个偶数节点，此时的ttail.next为otail，为了避免链表有环，所以需要断开
       * tttail.next=null

#### 8.2 程序代码

```java
class Solution {
    public ListNode oddEvenList(ListNode head) {
        if(head==null||head.next==null)
            return head;
        ListNode o=head,otail=o,ttail=t;
        while(ttail.next!=null){
            otail.next=ttail.next;
            otail=otail.next;
            if(otail.next!=null){
                ttail.next=otail.next;
                ttail=ttail.next;
            }
            else
                ttail.next=null; 
        }
        otail.next=t;
        return o;
        
    }
}
```



