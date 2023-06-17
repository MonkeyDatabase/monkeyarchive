---
layout: post
title: 反转链表
excerpt: "反转链表不涉及复杂的算法，但是有很多细节，本文将介绍如何反转链表"
date:   2021-01-03 18:04:00
categories: [code]
comments: true
---

## 定义

1. 反转链表即将整个链表倒转过来，如1->2->3->NULL反转为NULL<-1<-2<-3
2. 遍历整个链表时，将当前节点的next改为指向前一个元素，由于节点没有引用它前一个元素，所以要提起存储前一个节点。因为是从前向后遍历，更改完前一个元素之后缓存前一个元素，接下来遍历当前元素时就可以指向前一个元素，再缓存当前元素，接着遍历下去
3. 当遍历到NULL时，该元素的前一个元素就是反转链表的表头，而它前一个元素被缓存了

## 例题

### 1、反转链表

题目地址为[Leetcode 206](https://leetcode-cn.com/problems/reverse-linked-list/)

#### 1.1 解题思路

1. 本题就是基础的反转链表

#### 1.2 程序代码

```java
class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode pre=null,cur=head,temp=null;
        while(cur!=null){
            temp=cur.next;
            cur.next=pre;
            pre=cur;
            cur=temp;
        }
        return pre;
    }
}
```

### 2、K 个一组翻转链表

题目地址为[Leetcode 25](https://leetcode-cn.com/problems/reverse-nodes-in-k-group/)

#### 2.1 解题思路

1. 学过反转链表后，我们可以把反转链表的方法单独拿出来
2. 每次反转一个组，反转完成后认为后面的不够分组，拼接回去
   * 当下一次循环真的不够分组的时候可以直接返回
   * 当下一次循环够分组，再修改上一组指向这一组的指针
3. 假设目前的分组不是第一个分组，它反转后，需要被前一个分组的尾指针连接，还需要将当前分组的尾指针指向未分组链表的头部，为便于第一个分组统一处理，可以创建一个表头，这样也方便返回整个串
4. 结束条件为
   * 整个链表正好可以分成K长度的分组，这样最后的head应该为空
   * 如果不可以正好分，则最后剩下的一定不够分组，此时按照长度遍历长度不足时tail就已经为空

#### 2.2 程序代码

```java
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        ListNode start=new ListNode(0);
        start.next=head;
        ListNode pretail=start,tail=null;
        while(head!=null){
            tail=pretail;
            for(int i=0;i<k;i++){
                tail=tail.next;
                if(tail==null)
                    return start.next;
            }
            ListNode nextgroup=tail.next;
            ListNode[] listnodes=reverse(head,tail);
            head=listnodes[0];
            tail=listnodes[1];
            pretail.next=head;
            tail.next=nextgroup;
            head=nextgroup;
            pretail=tail;
        }    
        return start.next;    
    }

    public ListNode[] reverse(ListNode head,ListNode tail){
        ListNode pre=null,cur=head,temp=null;
        while(pre!=tail){
            temp=cur.next;
            cur.next=pre;
            pre=cur;
            cur=temp;
        }
        return new ListNode[]{tail,head};
    }
}
```

