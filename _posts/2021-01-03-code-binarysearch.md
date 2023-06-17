---
layout: post
title: 二分法
excerpt: "对于有序数组的查找，二分法效率较高，本文将介绍二分法的步骤与原理"
date:   2021-01-01 16:28:00
categories: [code]
comments: true
---

## 定义

1. 排序数组的所有数字形成一个窗口，记窗口的左右边界索引分别为left和right，分别指向左右边界的首个元素，即left和right指向的元素被包含在数组里
2. 查找过程
   * 计算中间节点的序号，index=(left+right/2)
   * 如果nums[mid]>target，说明target在[mid+1,right]之间
   * 如果nums[mid]<target，说明target在[left,mid-1]之间
   * 如果nums[mid]=target，说明当前target已找到，如果**无重复**元素则运行结束，如果有重复，则继续
     * 如果下次进入[left,mid-1]，最终返回的是重复值的左边界，该边界值等于target
     * 如果下次进入[mid+1,right]，最终返回的是重复值的右边界，该边界值等于target
3. 查找的终止条件为left>right，因为left和right作为边界的窗口中包含left和right，如果left==right，此时窗口内还有值，并没有穷尽

## 模板

```java
//此模板中为选重复元素的左边界
public int binanrySearch(int[] nums,int target){
    int i=0,j=nums.length,mid=0;
    while(i<=j){
        mid=(i+j)/2;
        if(nums[mid]>target)
            left=mid+1;
        else
            right=mid-1;
    }
}
```

## 例题

### 1、统计一个数字在排序数组中出现的次数

题目地址为[剑指Offer 53-I](https://leetcode-cn.com/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof/)

#### 1.1 解题思路

1. 此题为排序数组，适合二分法
2. 此题要统计出现的次数，可以先找到一个边界，再向下遍历

#### 1.2 程序代码

```java
class Solution {
    public int search(int[] nums, int target) {
        int i=0,j=nums.length-1,m=0,result=0;
        while(i<=j){
            m=(i+j)/2;
            if(nums[m]<target)
                i=m+1;
            else
                j=m-1;
        }
        for(;i<nums.length&&nums[i]==target;i++)
            result++;
        return result;
        
    }
}
```

### 2、0～n-1中缺失的数字

题目地址为[剑指Offer 53-II](https://leetcode-cn.com/problems/que-shi-de-shu-zi-lcof/)

#### 2.1 解题思路

1. 因为是有序数组，所以可以使用二分法
2. 如果没有缺失数字，则nums[i]==i，一旦出现缺失，这个位置以后的所有位置nums[i]!=i

#### 2.2 程序代码

```java
class Solution {
    public int missingNumber(int[] nums) {
        int left=0,right=nums.length-1,mid=0;
        while(left<=right){
            mid=(left+right)/2;
            if(nums[mid]==mid)
                left=mid+1;
            else
                right=mid-1;
        }
        return left;
    }
}
```

### 3、搜索旋转排序数组

#### 3.1 解题思路

1. 虽然数组被旋转后，整体不再有序，但是使用二分法的话，还是会有一半是单调的，判断它是否单调是通过low和high的值的大小，因为旋转后分成了两半各自有序

#### 3.2 程序代码

```java
class Solution {
    public int search(int[] nums, int target) {

        int low=0,high=nums.length-1;

        while(low<=high){
            int mid=(low+high)/2;
            if(nums[mid]==target)
                return mid;
            if(nums[low]<=nums[mid]){
                if(nums[low]<=target&&nums[mid]>target)
                    high=mid-1;
                else
                    low=mid+1;
            }else{
                if(nums[mid]<target&&nums[high]>=target)
                    low=mid+1;
                else
                    high=mid-1;
            }
        }

        return -1;
    }

    public int binarySearch(int[] nums,int target,int low,int high){
        if(low<=high){
            int mid=(low+high)/2;
            if(nums[mid]==target)
                return mid;
            if(nums[low]<=nums[mid]){
                if(nums[low]<=target&&nums[mid]>target)
                    return binarySearch(nums,target,low,mid-1);
                else
                    return binarySearch(nums,target,mid+1,high);
            }
            else{
                if(nums[mid]<target&&nums[high]>=target)
                    return binarySearch(nums,target,mid+1,high);
                else
                    return binarySearch(nums,target,low,mid-1);
            }
        }
        return -1;
    }
}
```

