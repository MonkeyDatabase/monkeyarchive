---
layout: post
title: final、finally、finalize的区别
excerpt: "Java中相近词辨析"
date:   2021-01-01 18:17:00
categories: [Java]
comments: true
---

## 学习笔记

### 1. final

1. final修饰类，类不能被继承
2. final修饰方法，方法不能被重写
3. final修饰变量，变量一旦被赋值不能再更改

### 2. finally

finally是异常处理的一部分，finally语句块无论try语句块是否有异常都会正常执行

```java
try{
    
}catch(Exception e){
    e.printStackTrace();
}finally{
    
}
```

### 3. finalize

finalize是Object的一个方法，用于垃圾回收过程被GC调用释放资源，但是finalize方法被调用后对象不一定被回收，有可能finalize执行后又不需要回收了，当真正回收对象时，由于finalize方法只能被执行一遍，不会再执行，此时资源没有正常释放，所以应避免使用finalize方法

