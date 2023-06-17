---
layout: post
title: 虚拟机栈
excerpt: "java采用了基于栈的虚拟机，而c/c++是采用基于寄存器的虚拟机"
date:   2020-06-05 17:11:00
categories: [JVM]
comments: true
---

## 虚拟机栈

虚拟机栈是为了Java代码运行而产生的。目前运行程序有两种方式，一种是基于寄存器虚拟机运行，一种是基于软件栈运行。而Java虚拟机栈就是基于软件栈的，具有栈的特性。

## 栈帧

栈帧是一种更细粒度的数据结构。

主要解决以下问题：

1. 方法参数存在哪？局部变量存在哪？返回值存在哪？
2. 嵌套调用其他方法时，返回原方法时如何返回？

| 栈帧 | 栈帧作用 | 
|:--------|:-------:|
| 局部变量表 | 局部变量表大小由编译器决定，编译时赋值大小。存储形参和局部变量 | 
| 操作数栈 | 用于存储运行时的操作数 | 
| 动态链接 | 方法区内所属类的方法集合中的指针，是直接地址 | 
| 返回地址 | 包含局部变量表开始指针、操作数栈当前指针、程序计数器 | 
| 附加信息(可选) | 存储额外的信息 |

## jclasslib

jclasslib可以查看字节码文件内的结构，其中包括General Information、Constant Pool、Interfaces、Fields、Methods、Attributes。

| Methods内容 | 作用 | 
|:--------|:-------:|
| init | 此类的构造方法 | 
| 自定义方法名 | 每声明一个方法就出现一个对应的Method | 
| clinit | clinit是静态块，是static{}编译之后生成的字节码，不是符号引用 | 

## javap -verbose  xxxx.class

{% highlight java%}
Classfile /E:/Workspace/gg/out/production/gg/HelloWorld.class
  Last modified 2020年6月14日; size 622 bytes
  SHA-256 checksum db532c7336734197f6eb472964be3600f33e5e94df6fa73aadba5108cc2a088c
  Compiled from "HelloWorld.java"
public class HelloWorld
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // HelloWorld
  super_class: #7                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 3, attributes: 1
Constant pool:
   #1 = Methodref          #7.#24         // java/lang/Object."<init>":()V
   #2 = Class              #25            // HelloWorld
   #3 = Methodref          #2.#24         // HelloWorld."<init>":()V
   #4 = Fieldref           #26.#27        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = Methodref          #2.#28         // HelloWorld.add:()I
   #6 = Methodref          #29.#30        // java/io/PrintStream.println:(I)V
   #7 = Class              #31            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               LocalVariableTable
  #13 = Utf8               this
  #14 = Utf8               LHelloWorld;
  #15 = Utf8               main
  #16 = Utf8               ([Ljava/lang/String;)V
  #17 = Utf8               args
  #18 = Utf8               [Ljava/lang/String;
  #19 = Utf8               helloWorld
  #20 = Utf8               add
  #21 = Utf8               ()I
  #22 = Utf8               SourceFile
  #23 = Utf8               HelloWorld.java
  #24 = NameAndType        #8:#9          // "<init>":()V
  #25 = Utf8               HelloWorld
  #26 = Class              #32            // java/lang/System
  #27 = NameAndType        #33:#34        // out:Ljava/io/PrintStream;
  #28 = NameAndType        #20:#21        // add:()I
  #29 = Class              #35            // java/io/PrintStream
  #30 = NameAndType        #36:#37        // println:(I)V
  #31 = Utf8               java/lang/Object
  #32 = Utf8               java/lang/System
  #33 = Utf8               out
  #34 = Utf8               Ljava/io/PrintStream;
  #35 = Utf8               java/io/PrintStream
  #36 = Utf8               println
  #37 = Utf8               (I)V
{
  public HelloWorld();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LHelloWorld;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class HelloWorld
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        11: aload_1
        12: invokevirtual #5                  // Method add:()I
        15: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
        18: return
      LineNumberTable:
        line 3: 0
        line 4: 8
        line 5: 18
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      19     0        args   [Ljava/lang/String;
            8      11     1     helloWorld   LHelloWorld;

  public int add();
    descriptor: ()I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: bipush        30
         2: ireturn
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       3         0     this     LHelloWorld;
}
SourceFile: "HelloWorld.java"

{% endhighlight %}

1. stack、locals的大小在编译时已经确定
2. slot即为插槽，用于存储数据
3. new对象需要四步：new dup invokespecial astore_1，所以new方法不是线程安全的
4. 非静态方法的局部变量表Index为0的位置存储的都是this指针

## 方法调用

每个线程有两个属性：局部变量表开始指针、操作数栈当前指针。
1. 当执行main方法时，局部变量表开始指针和操作数栈当前指针分别指向main方法的局部变量表和操作数栈的对应位置。
2. 在main方法调用add方法时，局部变量表开始指针和操作数栈当前指针都需要修改，指向add方法的局部变量表和操作数栈的对应位置。
3. 在add方法执行完成后，需要恢复现场，把局部变量表开始指针和操作数栈当前指针指回main方法开始指针。
4. 由于在mian方法在调用add方法前已经执行了部分代码，在执行完add方法之后，肯定需要将程序计数器恢复到调用add方法之前。

## JVM中的程序计数器

1. 程序计数器是用于记录程序执行位置，便于方法调用、进程切换等恢复现场。
2. 程序计数器由运行时数据区维护，不归线程管理
3. Windows中程序计数器指向的是内存地址，而在Java中指向的是起始值为0的字节码的Index。

## 恢复现场

1. 线程恢复局部变量表起始指针、操作数栈当前指针
2. 调用执行引擎恢复程序计数器

## 常用字节码指令

1. new 创建新的对象实例
2. dup 复制栈顶元素并重新入栈
3. invokespecial 只用于三种场景：调用实例构造方法、调用private方法、调用父类方法super().method
4. astore_1 将操作数栈顶引用型数据pop，存入第2个本地变量表插槽
5. invokestatic 调用静态方法
6. invokeinterface 调用接口方法，在运行时再确定一个实现此接口的对象
7. invokevirtual 调用虚方法
8. invokedynamic 在运行时动态解析出调用点限定符所引用的方法，调用该方法，Java1.7时推出，主要用于支持JVM上的动态脚本语言(如Groovy、Jython等)
9. bipush 将单字节常量值推送到栈顶
10. istore_1 把栈顶int型变量pop出来赋值给局部变量表Index为1的位置
11. iload_1 将局部变量表Index为1的位置的int型变量推送到栈顶
12. iadd 依序将栈顶两个数pop，再将运算结果push进操作数栈
13. ireturn int型返回，return主要干四件事：修改两个指针、修改程序计数器、将结果压入原来方法的操作数栈中、回收内存
14. getstatic 获取静态字段的值

## new流程

{% highlight java %}
         0: new           #2                  // class HelloWorld
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
{% endhighlight %}

先生成一个空对象压入操作数栈，复制栈顶元素并将新元素压入操作数栈，使得操作数栈中存有两个空对象，执行invokespecial将栈顶元素pop出来赋值给init方法局部变量表Index为0的this指针，此时main方法的操作数栈就只有一个空对象。在init方法执行完成之后，返回一个初始化好的对象覆盖掉，此时操作数栈中有一个初始化好的对象，之后将引用型变量pop存入局部变量表中。

上面的0号和4号指令都是先有个操作码后跟个操作数，这类指令JVM都是分两步执行，首先执行操作码，再执行方法。例如4号指令先执行invokespecial构造方法的执行环境，包括对this指针的赋值，因为方法执行时需要这些参数，环境构造完成之后，再执行方法。

## 虚拟机栈与方法区的关系

每个方法的动态链接指向了方法区中所属类的方法集合中的物理地址

## 虚拟机栈与堆的关系

生成的对象都存储在堆中，而虚拟机栈中的局部变量表及操作数栈中存储的是引用型数据，即指向对象的指针。
