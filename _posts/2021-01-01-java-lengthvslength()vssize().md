---
layout: post
title: length、length()、size()的区别
excerpt: "Java中相近词辨析"
date:   2021-01-01 18:29:00
categories: [Java]
comments: true
---

## 学习笔记

### 1. length

length是数组的一个属性，即数组的长度

```java
for(int i=0;i<arr.length;i++){
    System.out.println(arr[i]);
}
```

### 2. length()

length()是String对象的一个方法，用于获取字符串的长度，其内部具体实现为final的char数组，所以length方法实际上返回的是char数组的长度

```java
for(int i=0;i<str.length();i++){
    System.out.println(str.charAt(i));
}
```

### 3. size()

size()是Java中Collection\<E\>接口和Map\<K,V\>接口的方法，用于返回Collection中有效元素的个数或Map中有效键值对的个数

* Collection
  * Set
    * HashSet
    * SortedSet
  * List
    * ArrayList
    * LinkedList
    * Vector
      * Stack
  * Queue
* Map
  * SortedMap
  * ConcurrentMap
  * HashMap
  * TreeMap
  * HashTable
  * .......