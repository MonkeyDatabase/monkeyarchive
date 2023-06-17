---
layout: post
title: 逆向新手题-SimpleUnpack
excerpt: "攻防世界新手逆向题目-SimpleUnpack"
date:   2020-09-01 19:05:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 下载之后是一个没有扩展名的文件
2. 拖入Hex Editor，发现魔数为0x7f454c46，为ELF文件格式
3. 由题目可知这个文件应该是个加壳的ELF程序
4. 拖入Exeinfo中查看，NOT Win EXE - .o - ELF executable [ 64bit obj. Exe file - CPU : AMD x86-64] ，Detected UPX! ，所以这是一个加了upx壳的64位ELF程序
5. upx -d [文件名] 脱upx壳
6. readelf -a [文件名] |grep flag 查看elf文件结果并筛选和flag相关内容
7. 使用IDA启动远程调试
   * 上传IDA服务器端程序/dbgsrv/linux_server64到kali，添加执行权限
   * 启动IDA服务器端程序
   * IDA-Debugger-Run-Remote Linux Debugger，启动远程调试
   * 输入程序完整路径、父级路径、kali系统ip
8. 在远程调试时，蒙出来了答案，具体如下：
   * 使用单步调试，很快执行完成
   * 这时候，点击停止程序
   * IDA弹出来了个对话框，问存储哪些segments，我点击了All segments
   * 之后IDA就对存储下来的指令进行了分析，分析结果和OllyDbg的类似
   * 直接就看到一个字符串，里面写的是flag{Upx_1s_n0t_a_d3liv3r_c0mp4ny}

## 独立思考

### 1. upx壳是什么原理？为什么upx -d就可以脱壳？

平时常常听到upx壳是最简单的压缩壳，但是我却对这个壳的实现原理毫无理解。

UPX压缩过的可执行文件体积缩小50%~70%，减少磁盘占用空间、文件上传下载的时间和其他损耗。程序经过UPX压缩过的程序和程序库没有功能损失，和压缩之前一样可以正常运行。

> [UPX源码](https://github.com/upx/upx)
>
> src路径下有filter、lzma-sdk、stub三个文件夹和很多个* .h和 *.cpp文件

upx -d将运行到OEP时的程序从内存中dump出来，存储成了无壳程序。

### 2. PE和ELF分别如何脱壳？

PE文件可以使用OllyDbg进行调试脱壳，ELF文件采用IDA进行调试脱壳。

### 3. IDA远程调试，点击停止程序，下载下来的是什么东西？

在调试过程中点击退出程序，弹出了一个对话框，这个对话框的内容是：

目标程序即将退出。数据库只包含了临时调试segments，如果你不想存储一份快照的话它们将被删除，所以你是否要拍摄一个内存快照？

此时我点击了Yes，又跳出一个对话框，内容是：

IDA即将从被测试进程复制数据到数据库，你想要保存哪些segments？

有两个选项，分别是所有segments、Loader segments。

此时我点击了All segments，然后IDA就重新布局了，整个程序就像调试一个PE程序一样，只不过程序更可读，我从左侧函数列表双击main函数，在IDA View-A中点击F5反汇编跳转到Pseudecode-A窗口，就看到了一句 **if ( !strcmp(&s1, flag) )**，此时双击flag，就跳到了IDA View-A窗口的数据段部分，而且看到了flag{Upx_1s_n0t_a_d3liv3r_c0mp4ny}。

同样的操作，我对未脱壳的程序也执行了一遍，发现拍摄的快照，IDA重新加载快照后无法识别出函数名，而且各个函数都是sub_xxxx类似的可读性很差的状态，找不到main函数，而且各个函数里的可读性也很低。虽然View-Open subviews-Strings[Shift+F12]通过Ctrl+F检索所有字符串也可以找到字符串flag{Upx_1s_n0t_a_d3liv3r_c0mp4ny}，但是这种方法能成应该是因为upx没有压缩数据区吧，如果这个字符串是计算得到的，如果不脱壳，想找到应该会很难。

### 4. 有壳程序在运行时一定会在内存中经过运算恢复到加壳之前的样子吗？

一般压缩壳和加密壳都会恢复源程序，找到OEP可以从内存中dump出源程序，需要修复IAT导入表。

目前主要的**疑问**是VMP壳是否会恢复到加壳之前，在网上查找之后，发现除了修复IAT，还需要修复VM代码，所以我猜是VMP壳是不会恢复到加壳之前的样子，其中需要依赖VM代码逐步运行，降低了运行效率/

## **产生过的疑问**

1. upx壳是什么原理？为什么upx -d就可以脱壳？
2. 为什么upx脱壳之后，没有生成新文件，而是旧文件是脱壳后的文件？
3. IDA远程调试，点击停止程序，下载下来的是什么东西？
4. 有壳程序在运行时一定会在内存中经过运算恢复到加壳之前的样子吗？

