---
layout: post
title: 逆向新手题-logmein
excerpt: "攻防世界新手逆向题目-logmein"
date:   2020-09-14 10:13:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 拖入ExeInfo，查看无壳

   * NOT Win EXE - .o - ELF executable [ 64bit obj. Exe file - CPU : AMD x86-64 - OS/ABI: unspecified ] 
   *  -> Compiler : GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.4) 5.4.0 20160609

2. 直接拖入IDA进行分析，可以直接看到main函数，反汇编代码如下

   ```c
   void __fastcall __noreturn main(__int64 a1, char **a2, char **a3)
   {
     size_t v3; // rsi
     int i; // [rsp+3Ch] [rbp-54h]
     char s[36]; // [rsp+40h] [rbp-50h]
     int v6; // [rsp+64h] [rbp-2Ch]
     __int64 v7; // [rsp+68h] [rbp-28h]
     char v8[8]; // [rsp+70h] [rbp-20h]
     int v9; // [rsp+8Ch] [rbp-4h]
   
     v9 = 0;
     strcpy(v8, ":\"AL_RT^L*.?+6/46");
     v7 = 28537194573619560LL;
     v6 = 7;
     printf("Welcome to the RC3 secure password guesser.\n", a2, a3);
     printf("To continue, you must enter the correct password.\n");
     printf("Enter your guess: ");
     __isoc99_scanf((__int64)"%32s", (__int64)s);
     v3 = strlen(s);
     if ( v3 < strlen(v8) )
       wrong();
     for ( i = 0; i < strlen(s); ++i )
     {
       if ( i >= strlen(v8) )
         wrong();
       if ( s[i] != (char)(*((_BYTE *)&v7 + i % v6) ^ v8[i]) )
         wrong();
     }
     success();
   }
   ```

   * 首先输入字符串长度要大于v8长度，如果小于则回显错误并退出程序
   * 接下来循环检测s的每一个字符，当i大于v8长度时报错，所以s长度要和v8长度一致
   * v7初始为int64类型的28537194573619560LL，后来使用**(_BYTE *)&v7**取v7的地址所为指向BYTE型数据的指针，所以v7被看作BYTE类型的数组
   * **(_BYTE *)&v7 + i % v6**是移动指针，因为v6为常量7，所以使用了int64的前七个BYTE
   * **(char)(*((_BYTE *)&v7 + i % v6) ^ v8[i]) )**是把v7[i]与v8[i]进行按位或，之后作为char类型数据，与s[i]进行比较，如果相同则通过测试

3. 简化反汇编程序，如下：

   ```c
   int mian(){
       char[8] v8;
       strcpy(v8, ":\"AL_RT^L*.?+6/46");
       char[8] v7 = (char *)28537194573619560LL;
       char[8] s;
       printf("Welcome to the RC3 secure password guesser.\n");
       printf("To continue, you must enter the correct password.\n");
       printf("Enter your guess: ");
       scanf("%s",s);//s和v8一样长
       for(int i=0;i<strlen(v8);i++){
           if(s[i]!=(char)(v7[i]^v8[i])){
               printf("Incorrect password!\n");
               return 0;
           }
       }
       printf("You entered the correct password!\nGreat job!\n");
       return 0;
   }
   ```

4. 第一次解题程序，出现问题

   ```c
   #include<stdio.h>
   #include<string.h>
   #include<stdlib.h>
   
   int main(){
       long long v7=28537194573619560LL;
       
       char v8[8];
       strcpy(v8,":\"AL_RT^L*.?+6/46");   
       for(int i=0;i<strlen(v8);++i){
           printf("%c",(char)(*((char*)&v7 + i %7 ) ^ v8[i]));
       }
       
       printf("\n");
       system("pause");
       return 0;
   }
   
   //输出结果为ostd{fc
   //错误
   ```

5. 查看writeup后发现问题出在v8的复制上，：

   ```c
   #include<stdio.h>
   #include<string.h>
   #include<stdlib.h>
   
   int main(){
       __int64 v7=28537194573619560LL;
       
       //char v8[8];
       //strcpy(v8,":\"AL_RT^L*.?+6/46");   
       char v8[18]=":\"AL_RT^L*.?+6/46";
       for(int i=0;i<strlen(v8);++i){
           printf("%c",(char)(*(( char *)&v7 + i %7 ) ^ v8[i]));
       }
       
       printf("\n");
       system("pause");
       return 0;
   }
   //输出结果为RC3-2016-XORISGUD
   //正确
   ```

   问题出在了v8一开始进行了strcpy，将17字节复制进了v8原有的空间中，造成了缓冲区溢出

6. 证明：

   ```c
   #include<stdio.h>
   #include<string.h>
   #include<stdlib.h>
   
   int main(){
       __int64 v7=28537194573619560LL;
       
       char v8[8];
       strcpy(v8,":\"AL_RT^L*.?+6/46");   
       printf("%lld\n",v7);
   
       
       printf("\n");
       system("pause");
       return 0;
   }
   //打印了v7为3760283773249137228
   //而v8初始就有18B时，v7正确为28537194573619560
   //应该是v7在内存中在v8后面，当strcpy时，由于原v8空间不够，占用了v7原有的空间，导致运算出错
   ```

## 独立思考

IDA分析出来的数据长度并不可靠,需要自己进行判断,解题时避免自己出现缓冲区溢出。而且使用n快捷键可以在IDA中重命名函数，但是重命名变量时需要注意。ELF程序不一定要使用IDA远程动态调试，如果没有壳，可以直接在IDA中进行静态分析。

__int64在c语言中是一个数据类型。

BYTE不同于char，而是unsigned char，注意这些细节。