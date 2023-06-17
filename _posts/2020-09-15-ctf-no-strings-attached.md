---
layout: post
title: 逆向新手题-no-strings-attached
excerpt: "攻防世界新手逆向题目-no-strings-attached"
date:   2020-09-15 14:52:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 下载之后发现无后缀名，使用Hex editor打开，魔数为0x7f454c46，为ELF文件格式


2. 拖入Exeinfo查看

   ```
   NOT Win EXE - .o - ELF executable [ 32bit obj. Exe file - CPU : Intel 80386 - OS/ABI: unspecified ] 
    -> Compiler : GCC: (Ubuntu/Linaro 4.6.3-1ubuntu5) 4.6.3
   ```

3. 使用Linux系统下的file工具查看，发现未去除调试信息，运行可能会有辅助信息

   ```shell
   ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=c8d273ed1363a1878f348d6c506048f2354849d0, not stripped
   ```
   
4. 在Linux系统下运行，查看大致功能，发现报错崩溃

   ```shell
   welcome to cyber malware control software.
   Currently tracking 811437583 bots worldwide
   malloc.c:2389: sysmalloc: Assertion `(old_top == initial_top (av) && old_size == 0) || ((unsigned long) (old_size) >= MINSIZE && prev_inuse (old_top) && ((unsigned long) old_end & (pagesize - 1)) == 0)' failed.
   Aborted
   ```
   
5. 根据题目提示，该程序只要能够顺利运行就可以拿到flag，那么目前的报错经过检查之后是内存越界导致的,经过尝试找不到问题所在。

6. 使用IDA静态分析，发现decrypt函数计算出来一个字符串与输入字符串进行比较，只是该字符串比较特殊，每个字符是**wchar_t**类型，长度要长于char类型，包含许多char无法表示的字符。

7. 尝试使用IDA静态定位函数内存地址(options-General-Line prefixes)，然后使用Remote Linux Debugger远程动态调试，直接断在decrypt函数刚执行完，可以直接从内存中读取。然而本程序存在问题，无法执行到目标函数就会Abort

8. 分析decrypt函数

   ```c
   wchar_t *__cdecl decrypt(wchar_t *s, wchar_t *a2)
   {
     size_t v2; 
     signed int v4; 
     signed int i; 
     signed int v6;
     signed int v7; 
     wchar_t *dest; 
   
     v6 = wcslen(s);
     v7 = wcslen(a2);
     v2 = wcslen(s);
     dest = (wchar_t *)malloc(v2 + 1);
     wcscpy(dest, s);
     while ( v4 < v6 )
     {
       for ( i = 0; i < v7 && v4 < v6; ++i )
         dest[v4++] -= a2[i];
     }
     return dest;
   }
   ```

   * v6为s长度，v7为v2长度
   * 首先将s字符串拷贝到dest
   * 之后进入一个循环，每次循环中又嵌套一个循环，让dest[]减去a2[],可以推测出v7小于v6，因为内层每一次循环v4也增加了1，而内层循环结束条件是i<v7，要使双层循环有意义，那么外层循环次数一定大于v7
   * 可以看出是s按照a2的长度为周期，对应位置减去a2，最终即为所得。
   * 所以目前的难度在于提取出传入decrypt函数的s和dword_8048A90两个字符串内容

## 别人的解题过程

1. wchar_t类型长度为16位或32位，由于有大量的0，所以不能用char数组导出，否则0x14010000本来是一个wchar_t，如果当作char，第三个char就是字符串结束符\0了，会导致读取出现问题。

2. IDA可使用shift+E快速到处选中的变量，导出时需注意字符串数组以0作结束符，而新声明的字符串会自动填充，所以提取时需要去掉目前有的0

3. 编写程序

   ```c
   #include<stdio.h>
   #include<stdlib.h>
   int main(){
       unsigned char s[] =
           {
              58,  20,   0,   0,  54,  20,   0,   0,  55,  20, 
               0,   0,  59,  20,   0,   0, 128,  20,   0,   0, 
             122,  20,   0,   0, 113,  20,   0,   0, 120,  20, 
               0,   0,  99,  20,   0,   0, 102,  20,   0,   0, 
             115,  20,   0,   0, 103,  20,   0,   0,  98,  20, 
               0,   0, 101,  20,   0,   0, 115,  20,   0,   0, 
              96,  20,   0,   0, 107,  20,   0,   0, 113,  20, 
               0,   0, 120,  20,   0,   0, 106,  20,   0,   0, 
             115,  20,   0,   0, 112,  20,   0,   0, 100,  20, 
               0,   0, 120,  20,   0,   0, 110,  20,   0,   0, 
             112,  20,   0,   0, 112,  20,   0,   0, 100,  20, 
               0,   0, 112,  20,   0,   0, 100,  20,   0,   0, 
             110,  20,   0,   0, 123,  20,   0,   0, 118,  20, 
               0,   0, 120,  20,   0,   0, 106,  20,   0,   0, 
             115,  20,   0,   0, 123,  20,   0,   0, 128,  20, 
               0,   0
           };
       unsigned char a2[] =
           {
               1,  20,   0,   0,   2,  20,   0,   0,   3,  20, 
               0,   0,   4,  20,   0,   0,   5,  20,   0,   0, 
           };
       int i=0;
       while(i<152){
                    s[i]-=a2[i%20];
                    //跳过中间的0                 
                    if(s[i]!=0)
                    	printf("%c",s[i]);
                    i++;
                    }
       system("pause");
       return 0;
       }
   //输出结果为9447{you_are_an_international_mystery}
   ```

   