---
layout: post
title: JVM-Java线程模型的疑问
excerpt: "由于测试虚拟机栈OutOfMemoryError时，产生了3000+线程，对Java线程模型产生了很大疑问"
date:   2020-07-20 17:21:00
categories: [Question]
comments: true
---

## 一、代码

```java
package OOM;

/**
 * VM Options: -Xss2M
 * 由于Hotspot虚拟机禁用了栈深度扩展
 * 则引发栈OOM，只可能发生在申请新栈
 * 即申请新线程时
 *
 * 很难触发，因为系统假死
 */
public class JavaVMStackOOM {
    public volatile static int threadCount=0;

    public void dontStop(){
        while(true){

        }
    }

    public void stackTestByTread(){
        while(true){
            System.out.println("you:"+threadCount);
            Thread thread=new Thread(new Runnable() {
                @Override
                public void run() {
                        threadCount++;
                        dontStop();
                }
            });
            thread.start();
        }
    }

    public static void main(String[] args) {
        JavaVMStackOOM oom=new JavaVMStackOOM();
        oom.stackTestByTread();
    }
}
```

从控制台输出可以看出，这个程序开出了数千个线程。

## 二、疑问点

《深入理解Java虚拟机》书中写道“线程有三种实现方式：使用内核线程实现、使用用户线程实现、使用用户线程加轻量级进程混合实现”、“JDK1.3起，主流平台上的主流商用虚拟机的线程模型都被替换为基于操作系统原生线程模型来实现，即采用1：1的线程模型”、“以Hotspot为例，它的每一个Java线程都是直接映射到一个操作系统原生线程来实现的，而且中间没有额外的间接结构，所以Hotspot虚拟机自己是不会去干涉线程调度的”。



1. 本机是六核、12线程的CPU，那么内核级线程数应该是多少？
2. 为什么采用1：1线程模型，Java程序可以申请如此多的线程数？