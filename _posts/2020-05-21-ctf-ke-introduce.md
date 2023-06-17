---
layout: post
title: 逆向破解入门-第一课
excerpt: "程序有没有加壳？加了什么壳？"
date:   2020-05-21 14:00:00
categories: [CTF]
comments: true
---

## 简介

一个程序编译出来之后是没有壳的状态。不同编译器编译出的无壳程序是不同的，先认识不同编译器编译出来的无壳程序，再研究加壳程序会很容易。

## ollydbg

### 窗口

* 反汇编窗口
* 寄存器窗口
* 数据窗口
* 堆栈窗口

### 功能

* l：程序运行和插件加载的日志
* e: 模块加载信息
* m：内存信息
* t：线程信息
* w:窗口
* h:句柄

## ExeInfoPE

### 使用

把软件拖进ExeInfoPE，诊断信息会显示使用的编译器及有没有加壳。

### 区段

点击EP Section右侧的...会显示区段信息(OD的m也可以看)

## 无壳程序

程序编译之后是无壳的，没有压缩过，内部是可以修改的，可以进行diy。	

### VC 6.0

1. 入口点代码是固定的
2. 入口API也是相同的
3. 区段有四个：.text, .data, .rdata, .rsrc

唯一不同的可能是push的地址是不同的

### VS 2008/2013

1. 特征只有两个：call, jmp
2. 区段有五个：.text, .rdata, .data, .rsrc, .reloc(重定位区段)

### 易语言

#### 非独立编译

1. 入口特征和模块特征都有krnln.fnr

#### 独立编译

1. 搜索字符串，检查常规的易语言自带的字符串
2. 进call里看一下有没有调用一堆核心库jmp

### Delphi

1. 区段有CODE, DATA, BSS, .idata, tls, .rdata, .reloc, rsrc

### BC++

1. BC++的入口是一个很大的jmp
2. 区段有很多.text, .data, .tls, .rdata, .idata, .didata, .edata, rsrc, reloc

### .NET

1. .NET程序拖到OD里可能就直接运行了，一方面可能是有tls，默认勾选了break on tls
2. .NET程序需要用特定的插件断点分析
3. .NET用OD查看模块可以看到调用了.NET的库

### PB/QT

1. 用OD查看模块就可以看到相应的库

## 有壳程序

### 查壳软件

1. PEiD已经停止更新了，各种特征库已经不能用了
2. ExeinfoPE还在更新

### aspack

1. 入口是pushad，call，jump

### upx

1. 点区段信息，一般只有三个区段，区段名可以随便改

### Themida

1. 区段信息，第一个是空的，.rsrc, .idata，还有两个任意名称的区段

### VMP

1. 区段信息，保留代码段和数据段，添加了三个段
2. 前面一大段跳转，入口pushad，后边跳向oep(程序入口点)

### Shielden

1. 区段名称有sedata，但名称可以改
2. Ctrl+A分析代码，F7单步步入几次，就可以看到字符串ascii“Safengine Shield”
