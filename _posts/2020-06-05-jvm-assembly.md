---
layout: post
title: jvm为什么能跨平台
excerpt: "Intel汇编 or AT&T汇编 or ARM汇编"
date:   2020-06-05 09:00:00
categories: [JVM]
comments: true
---

## Java工作流程

1. 程序员编写Java代码，存储格式为.java文件
2. 通过javac命令编译成字节码文件，存储格式为.class文件
3. 字节码文件在JVM上运行，由类加载器子系统加载到运行时数据区的方法区
4. Windows平台进行Intel汇编，Unix平台采用AT&T汇编，移动平台采用ARM汇编

## Intel汇编 vs AT&T汇编

| 特性 | Intel汇编 | AT&T汇编 |
|:--------|:-------:|:--------:|
| 派系 | Windows派系 | Unix派系 |
| 编译器 | VC | GCC |
| 寄存器 | rax,rbx,rcx,rdx | %rax,%rbx,%rcx,%rdx |
| 指令 | mov eax,8 | movl $8,%eax |

## class相关概念

1. class文件：java文件编译之后的字节码文件，存储在硬盘上
2. class content：类加载器将class文件加载到内存，与class文件的区别仅仅是存储位置不同，内容相同
3. class对象：类加载器基于JVM规范对class content进行解析，解析完成之后生成class对象，存储到方法区
4. 对象：new指令产生的对象，存储在堆中