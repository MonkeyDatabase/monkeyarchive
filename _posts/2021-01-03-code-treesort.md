---
layout: post
title: 堆排序
excerpt: "对于大数据量的TOP-K问题，最方便的算法就属堆排序，每次选出一个最大/最小的放到最终位置上，本文将介绍堆排序的用法"
date:   2021-01-03 21:28:00
categories: [code]
comments: true
---

## 定义

1. 堆就是指完全二叉树
   * 大顶堆指每个节点的值既大于左孩子的值，又大于右孩子的值，进而根节点是最大值
   * 小顶堆指每个节点的值既小于左孩子的值，又小于右孩子的值，进而根节点是最小值
2. 对于数组存储类型的堆，序号从0开始：
   * 序号为i的节点，其左孩子的序号为2i+1，右孩子的序号为2(i+1)
   * 序号为i的节点，其父节点的序号为i/2向下取整
3. 堆调整(以大顶堆为例)
   1. 将当前节点的值缓存
   2. 如果当前节点的数值
      1. 如果不小于左孩子或右孩子，直接返回
      2. 如果小于左孩子或右孩子，则选其中较大者节点与当前节点交换数据，并将该较大者节点看作新的当前节点，继续向下调整，如果到达叶子节点，则返回
4. 堆排序
   1. 先建堆，对于每个非叶子节点从后往前依次进行堆调整
   2. 选择根节点输出，将最后一个节点的数值与根节点交换，去除最后一个节点，在剩余节点中对根节点进行堆调整，调整完成后又恢复大顶堆

## 模板

```java
public void adjustHeap(int[] nums,int parent,int length){
    int temp=nums[parent];
    int lchild=2*parent+1;
    while(lchild<length){
        int rchild=lchild+1;
        if(rchild<length&&nums[rchild]>nums[lchild]){
            lchild=rchild;
        }
        if(temp>nums[lchild])
            break;
        nums[parent]=nums[lchild];
        
       parent=lchild;
       lchild=2*parent+1;
    }
    nums[parent]=temp;
}

public void heapify(int[] nums){
    for(int i=(nums.length-1)/2;i>=0;i--){
        adjustHeap(nums,i,nums.length);
    }
}

public void heapSort(int[] nums){
    heapify(nums);
    for(int i=nums.length-1;i>=0;i--){
        int temp=nums[0];
        nums[0]=nums[i];
        nums[i]=temp;
        adjustHeap(nums,0,i);
    }
    System.out.println(Arrays.toString(nums));
}
```

## 例题

### 1、数组中的第K个最大元素

题目地址为[Leetcode 215](https://leetcode-cn.com/problems/kth-largest-element-in-an-array/)

#### 1.1 解题思路

1. 本题就是一个排序的题目，可以采用堆排序，而且只需建完堆后提取K次数字，因为这道题不关心数字是否重复

#### 1.2 程序代码

```java
class Solution {
    public int findKthLargest(int[] nums, int k) {
        for(int i=(nums.length-1)/2;i>=0;i--){
            adjustHeap(nums,i,nums.length);
        }
        
        for(int i=nums.length-1;i>=0;i--){
            int temp=nums[0];
            nums[0]=nums[i];
            nums[i]=temp;
            adjustHeap(nums,0,i);
            k--;
            if(k==0)
                return nums[i];
         }
        return 0;
    }
    
    public void adjustHeap(int[] nums,int parent,int length){
        int temp=nums[parent];
        int lchild=2*parent+1;
        while(lchild<length){
            int rchild=lchild+1;
            if(rchild<length&&nums[rchild]>nums[lchild])
                lchild=rchild;
            if(temp>=nums[lchild])
                break;
            nums[parent]=nums[lchild];

            parent=lchild;
            lchild=2*lchild+1;
        }
        nums[parent]=temp;
    }
}
```
