---
layout: post
title: 类加载器
excerpt: "类加载器用于将类加载到内存中，解析出class对象存储到方法区。"
date:   2020-06-15 9:00:00
categories: [JVM]
comments: true
---

## 类加载过程

### 加载阶段

在硬盘上查找并通过IO读取字节码文件(类使用时才加载)

### 连接阶段

执行校验、准备、解析步骤

#### 校验阶段

校验字节码文件的正确性(重要但不必须，可通过-Xverifynone关闭)

1. 校验文件格式是否是class文件格式
2. 校验元数据是否符合Java规范
3. 校验字节码语义是否合法
4. 校验符号引用

#### 准备阶段

1. 为静态变量分配内存空间
2. 赋予静态变量默认值

> 赋默认值不是赋编码时程序员指定的值，数值型数据默认值一般为0，引用型数据默认值一般为null。动态变量是在堆的对象中的实例数据区进行存储，在实例化对象时分配空间。

#### 解析阶段

将符号引用转化为直接引用，该阶段会把一些静态方法(符号引用 如main())替换为指向数据所在内存的指针或句柄等(直接引用)，这就是静态链接。

1. 静态链接是类加载时完成
2. 动态链接是类运行时完成，如栈帧中的动态链接

> 符号引用在字节码文件中已经定义好了，符合Java规范，与虚拟机内部内存布局无关，在不同的虚拟机中都可正常运行。而直接引用与内存布局是有关的，不同虚拟机对同样符号引用翻译出来的直接引用格式可能不同。

> javap -v xxx.class 指令执行后的常量池(Constant Pool)内存储了大量的符号引用。

### 初始化阶段

为静态变量赋程序员指定的值，执行静态代码块。

### 使用阶段

### 卸载阶段

## 类加载器

> 核心类和扩展类在JVM启动时一次性全部加载

### 启动类加载器

负责加载支撑JVM运行的位于JDK/JRE/lib目录下的核心类库，比如rt.jar,charsets.jar等

### 扩展类加载器

负责加载支撑JVM运行的位于JDK/JRE/ext扩展目录下的类库

### 应用程序类加载器

负责加载ClassPath路径下的类库，主要就是加载自己写的类

### 自定义加载器

负责加载自定义路径下的包

{% highlight java %}
public class EurekaApplication {
    public static void main(String[] args){
        //SpringApplication.run(EurekaApplication.class,args);
        System.out.println(String.class.getClassLoader());//这句和其他两句不一样，是因为启动类加载器是c语言编写的，访问不到ClassLoader，返回为null，无法像另外两个getClass
        System.out.println(com.sun.crypto.provider.DESKeyFactory.class.getClassLoader().getClass().getName());
        System.out.println(EurekaApplication.class.getClassLoader().getClass().getName());
    }
}
{% endhighlight %}

{% highlight java %}
null
sun.misc.Launcher$ExtClassLoader
sun.misc.Launcher$AppClassLoader
{% endhighlight %}
