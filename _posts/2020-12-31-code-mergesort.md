---
layout: post
title: 归并排序
excerpt: "在刷算法题的过程中，常常遇到排序，归并排序分而治之的方法十分有效，可以大大避免超时的问题，因此本文记录一些归并排序相关的知识"
date:   2020-12-31 14:18:00
categories: [code]
comments: true
---

## 定义

1. 归并排序是建立在归并操作的基础上的一种有效的排序算法，该算法采用分治法。
3. 时间复杂度：一般能在O(nlongn)的时间解决问题，代价是需要额外的存储空间
3. 算法步骤
   1. 申请空间，使其大小为两个排序序列之和，该空间用来存放排序后的序列
   2. 设定两个指针，最初位置分别指向两个已经排序序列的起始位置
   3. 比较两个指针所指向的元素，选择较小的放入合并空间，并将指针向后移动
   4. 重复第3步，直至某一个指针走到序列尾
   5. 将未到序列尾的序列中的剩余元素直接复制到合并空间的尾部
4. 简化计算技巧
   * 当第一个序列的最后一个值即最大值值小于第二个序列的第一个值即最小值，则无需进行归并，这两个序列拼接起来必定有序

## 代码

```java
public class MergeSort{
    public static void main(String[] args){
        int[] arr = {1,3,123,56,64,4,52,45,56,3};
        int[] tmp = new int[arr.length];
        mergeSort(arr,tmp,0,arr.length-1);
    }
    
    public int[] mergeSort(int[] arr,int[] tmp,int low,int high){
        if(low<high){
            int mid=(low+high)/2;
            mergeSort(arr,tmp,low,mid);
            mergeSort(arr,tmp,mid+1,high);
            if(arr[mid]>arr[mid+1])
            	merge(arr,tmp,low,mid,high);
        }
    }
    
    public int[] merge(int[] arr,int[] tmp,int low,int mid,int high){
        int i=0,j=low,k=mid+1;
        while(j<=mid&&k<=high){
            if(arr[j]>arr[k])
                tmp[i++]=arr[j++];
            else
                tmp[i++]=arr[k++];
        }
        while(j<=mid)
            tmp[i++]=arr[j++];
        while(k<=high)
            tmp[i++]=arr[k++];
        for(int t=0;t<i;t++)
            arr[low+t]=tmp[t];
    }
}
```



## 例题

### 1、数组中的逆序对

题目地址为[剑指Offer 51](https://leetcode-cn.com/problems/shu-zu-zhong-de-ni-xu-dui-lcof/)

#### 1.1 解题思路

1. 采用分而治之的归并排序算法
2. 假设进行到将两个数组a和b归并(a在前，b在后)，且各自数组内部有序，即内部逆序为0，则统计逆序只需用两个指针分别指向a和b中的值，如果出现a的某个值大于b中的某个值，则需要把b中的值提前到a指针所指向的元素元素的前面，则此时说明a数组当前指针后面所有的值都要大于b中当前指针指向的值，则此时逆序为(mid-a中指针+1)

#### 1.2 程序代码

```java
class Solution {
    int count=0;
    public int reversePairs(int[] nums) {
        int[] tmp=new int[nums.length];
        mergesort(nums,tmp,0,nums.length-1);
        return count;
    }

    public void mergesort(int[] nums,int[] tmp,int low,int high){
        if(low<high){
            int mid=(low+high)/2;
            mergesort(nums,tmp,low,mid);
            mergesort(nums,tmp,mid+1,high);
            if(nums[mid]>nums[mid+1])
                merge(nums,tmp,low,mid,high);
        }
    }

    public void merge(int[] nums,int[] tmp,int low,int mid,int high){
        int i=0,j=low,k=mid+1;
        while(j<=mid&&k<=high){
            if(nums[j]<=nums[k])
                tmp[i++]=nums[j++];
            else{
                tmp[i++]=nums[k++];
                count+=(mid-j+1);
            }

        }
        while(j<=mid)
            tmp[i++]=nums[j++];            
        while(k<=high)
            tmp[i++]=nums[k++];
        for(int t=0;t<i;t++)
            nums[t+low]=tmp[t];
    }
}
```

### 2、合并两个排序的链表

题目地址为[剑指Offer 25](https://leetcode-cn.com/problems/he-bing-liang-ge-pai-xu-de-lian-biao-lcof/)

#### 2.1 解题思路

1. 由于是合并两个排序的链表，和归并排序很类似

2. 底层存储结构采用了链表，而不是数组所以插入操作的时间复杂度为O(1)

   * 不用像原本的归并排序一样占用大量的辅助空间

   * 不用复制每一个节点到一个新的位置，只需将另一个链表按顺序插入当一个链表中，且能保持有序

   * 不用像数组存储结构时循环依次复制，而只需用指针指向剩余部分的头

#### 2.2 程序代码

```java
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        ListNode head=new ListNode(0),p=head;
        while(l1!=null&&l2!=null){
            if(l1.val<l2.val){
                p.next=l1;
                l1=l1.next;
            }
            else{
                p.next=l2;
                l2=l2.next;
            }
            p=p.next;
        }
        p.next = (l1!=null) ? l1:l2;
        return head.next;
    }
}
```

### 3、合并两个有序数组

题目地址为[Leetcode 88](https://leetcode-cn.com/problems/merge-sorted-array/)

#### 3.1 解题思路

1. 此题两个数组都为有序，使用归并排序最快
2. 本题有一个巧妙的点就是不能用辅助空间，要把第二个数组合并到第一个数组中。我们学过的归并排序是从低排到高，本题中第一个数组高地址存储的全为0，因此此题可以采用从高地址排到低地址，而且只要最后对第二个数组判断有没有空

#### 3.2 程序代码

```java
class Solution {
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int i=m+n-1,j=m-1,k=n-1;
        while(j>-1&&k>-1){
            if(nums1[j]>nums2[k])
                nums1[i--]=nums1[j--];
            else
                nums1[i--]=nums2[k--];
        }
        while(k>-1)
            nums1[i--]=nums2[k--];
    }
}
```

