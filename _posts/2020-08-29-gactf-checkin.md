---
layout: post
title: GACTF-Reverse-Checkin
excerpt: "本文为GACTF-Reverse-Checkin的解题过程"
date:   2020-08-29 09:26:00
categories: [CTF]
comments: true
---

## 解题过程

1. 题目下载后压缩包解压有一个Checkin.exe文件
2. 使用Exeinfo进行查壳，发现为GCC编译的32为可执行程序
3. 初次执行，发现先进行了大量运算，最后输出了个"key:"
4. 使用OllyDbg调试时，弹出来"FATAL ERROR: Failed to create process (C:\Users\Diablo\AppData\Local\Temp\ocrB7DC.tmp\bin\ruby.exe): 50"，并没有输出"key:"，所以目前没办法动态调试
5. 由于它一开始进行了大量运算，所以拖进kali用binwalk提取文件试一试，结果提取出来了个C608.7z和一个C608文件，但是两个文件均打不开
6. 之后这道题没做出来

后来的解题方法：

* 运行Checkin.exe，待输出"key:"之后，在C:\Users\Diablo\AppData\Local\Temp\中查找名称带ocr的最近生成的文件夹

* 打开该路径，发现有bin、lib、src三个文件夹

* src文件夹下有一个Checkin.rb文件，应该为程序源码

  ```ruby
  require 'openssl'  
  require 'base64'  
  
   
  def aes_encrypt(key,encrypted_string)
  	aes = OpenSSL::Cipher.new("AES-128-ECB")
  	aes.encrypt
  	aes.key = key
  	cipher = aes.update(encrypted_string) << aes.final
  	return Base64.encode64(cipher) 
  end
  
  //输入flag，通过gets.chomp存储到flag
  print "Enter flag: "
  flag = gets.chomp
  
  key = "Welcome_To_GACTF"
  cipher = "4KeC/Oj1McI4TDIM2c9Y6ahahc6uhpPbpSgPWktXFLM=\n"
  
  text = aes_encrypt(key,flag)
  if cipher == text
  	puts "good!"
  else
  	puts "no!"
  end
  ```

* 可以看出flag经"Welcome_To_GACTF"密钥加密，采用Base64输出，密文为"4KeC/Oj1McI4TDIM2c9Y6ahahc6uhpPbpSgPWktXFLM=\n"

* 所以可以采用[AES在线加解密](http://tool.chacuo.net/cryptaes/)进行解密，但是需要把密文最后的\n给删除才能解密

* 解密得到flag为GACTF{Have_a_wonderful_time!}

### API列表

| API名称                | 作用                                                         |
| ---------------------- | ------------------------------------------------------------ |
| CreateProcessInternalW | 在创建进程中被CreateProcessA调用                             |
| CreateProcessA         | 创建一个新进程和它的主进程，新进程在调用进程的上下文中执行， |



## 独立思考

### 1. 为什么用OllyDbg会崩溃，而直接双击执行可以执行？

发现使用原版OllyDbg是可以调试的，但是吾爱破解论坛的OllyDbg是会创建进程失败。

Checkin.exe中途在C:\Users\Diablo\AppData\Local\Temp\生成了许多文件其中包括ruby.exe，它创建进程启动ruby.exe，使用ruby.exe去调用src/Checkin.rb。

### 2. AppData\Local\Temp\路径下存储的什么内容？

Appdata位于系统盘-用户目录下：

* Local：本地保存文件，其下的Temp文件中存储本地临时文件
* LocalLow：共享数据存放文件
* Roaming：保存应用程序运行后的数据信息，如果删除应用程序运行配置信息会丢失

### 3. 为什么Ruby程序追踪不到核心代码？

使用原版OllyDbg通过F8、F7、F9逐步缩小范围，发现Checkin.exe创建了一个新进程用于使用rube.exe执行Checkin.rb代码，这部分代码全部位于大地址处，而不是位于Checkin.exe的代码中。而且，在打印出"key:"之后，Checkin.exe被OllyDbg暂停，此时我在小黑框中输入flag的值，立刻就得到了"good!"或者"no!"，说明ruby代码并不受调试影响，自然我们观察和控制的代码也就不是真实有效的加解密代码。

所以以后遇到这种情况，要根据寄存器中字符串和堆栈列表定位到它真实调用的程序，如本题中它在程序执行时将真实代码释放到了Appdata/Local/Temp文件夹下，ruby语言类似于python，是一种解释型语言，所以我们可以找到它的源码，继而解出题目。

### 4. 为什么解密的时候要把\n删除，为什么它就可以加密后让带\n的字符串与密文相等？

1. 字符串中的\n是换行符，部分Base64库仍遵守RFC822规定，编码每76个字符，需要加上一个回车换行。

2. 由源码可知最后进行了Base64编码，而Base64编码结尾会用=来进行填充到合适长度，所以\n是多余的。

## 产生过的疑问

1. 为什么用OllyDbg会崩溃，而直接双击执行可以执行？
2. C:\Users\Diablo\AppData\Local\Temp\路径下存储的什么内容？
3. 为什么Ruby程序追踪不到核心代码？
4. 为什么解密的时候要把\n删除，为什么它就可以加密后让带\n的字符串与密文相等？

