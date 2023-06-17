---
layout: post
title: 快速排序
excerpt: "对于大数据量的排序中，堆排序、快排序、归并排序相对较快，本文将介绍快速排序的思想"
date:   2021-01-04 11:56:00
categories: [code]
comments: true
---

## 定义

1. 在快速排序种每次对一个区间内进行排序，在左边界left和右边界right中间的某个位置确定一个pivot的最终位置，pivot的左侧的值均小于它，pivot右侧的值均大于它，从而pivot将这个区间分成了两部分，[left,pivot-1]和[pivot+1,right]两个区间分别再次进行快排序
2. 分区间函数partition()为了让pivot左侧的值均小于它，右侧的值均大于它
   1. 选区间的第一个数，即left指向的数值为pivot，进行缓存
   2. 设计一个循环，循环条件为right指针大于left指针
      * 将right指针不断向左移**且**保证right指针大于left指针，**直到**right指向的值**小于**pivot，移动停止，将right指针指向的值赋值给left指针指向的位置
      * 将left指针不断右移且保证right指针大于left指针，**直到**left指向的值**大于**pivot，移动停止，将left指针指向的值赋值给right指针指向的位置
   3. 当循环结束时，left指针和right指针相遇，此时将low作为分割点返回
3. 递归函数quickSort()
   1. 调用分区函数，并获取分割点
   2. 递归调用自身，左区间[left,pivot-1]
   3. 递归调用自身，右区间[pivot+1,right]

## 例题

### 1、排序数组

题目地址为[Leetcode 912](https://leetcode-cn.com/problems/sort-an-array/)

#### 1.1 解题思路

1. 本题没有任何要求，只是考察排序算法升序排列

#### 1.2 程序代码

```java
class Solution {
    public int[] sortArray(int[] nums) {
        quickSort(nums,0,nums.length-1);
        return nums;
    }

    public void quickSort(int[] nums,int low,int high){
        if(low<high){
            int pivot=partition(nums,low,high);
            quickSort(nums,low,pivot-1);
            quickSort(nums,pivot+1,high);
        }
    }

    public int partition(int[] nums,int low,int high){
        int temp=nums[low];        
        while(low<high){
            while(low<high&&nums[high]>=temp)
                high--;
            nums[low]=nums[high];
            while(low<high&& nums[low]<=temp)
                low++;
            nums[high]=nums[low];
        }
        nums[low]=temp;
        return low;
    }
}
```

