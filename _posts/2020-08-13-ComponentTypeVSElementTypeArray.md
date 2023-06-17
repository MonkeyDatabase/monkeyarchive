---
layout: post
title: 数组的组件类型和元素类型有什么区别和联系？
excerpt: "虚拟机执行子系统包括类文件结构、虚拟机类加载机制、虚拟机字节码执行引擎。在虚拟机类加载机制ClassLoading的加载阶段Loading对于数组类型和非数组类型有不同的加载办法，而对于数组类型的讲解部分讲到了组件类型和元素类型，这两者有什么区别和联系呢？为什么在虚拟机执行子系统的虚拟机类加载子系统的加载阶段讲解？"
date:   2020-08-13 08:36:00
categories: [JVM]
comments: true
---

## 学习笔记

### 一、虚拟机如何Loading

**Loading**阶段是**Class Loading**过程的一个阶段。

在Loading阶段，JVM需要完成三件事：

1. 通过一个类的全限定名来获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的入口

### 二、非数组类型加载

相比于Class Loading的其他阶段，**非数组类型**的Loading(准确来说是Loading中的第一件事情：通过一个类的全限定名获取定义这个类的二进制字节流)是开发人员可控最强的。

非数组类型Loading的过程可以采用虚拟机内置的引导类加载器，也可以由用户自定义的类加载器去完成，程序员可以通过重写**findClass()**或**loadClass()**方法，实现自己的类加载器，从而按照自己的需求实现Class Loading获取运行代码的动态性。

### 二、数组类型加载

数组类型不同于非数组类型，数组类本身不通过类加载器创建，它是由JVM直接在内存中动态构造出来的。

#### 1. 元素类型

Element Type元素类型，是指数组**去除所有维度**之后剩下的单个个体的类型。

#### 2. 组件类型

Component Type组件类型，是指数组**去除一个维度**之后剩下的单个个体的类型。

1. 当组件类型是引用类型时，递归Loading过程来加载这个组件类型，并将该数组C标记到Loading组件类型的类加载器的类名称空间上
2. 当组件类型不是引用类型，JVM将会把该数组C标记为与引导类加载器关联
3. 数组类的可访问性和它的组件类型一致！且当组件类型不是引用类型时，它的数组类型的可访问性**默认**为public，可被所有的类和接口访问。

## 独立思考

#### 1. 如何通过全限定名获取定义此类的二进制字节流？

JVM规范中只规定了Loading时要完成三件事情：

1. 通过类的全限定名获取二进制字节流
2. 将二进制字节流代表的静态存储结构转化为动态存储结构
3. 在堆区生成一个java.lang.Class对象作为这个类在方法区中各种数据的访问入口

但是JVM规范并没有对这三件事规定具体每一步如何做，所以如何实现这三件事由虚拟机发行商和程序员自行设计实现。

例如获取二进制字节流并没有规定**从哪里获取**和**如何获取**，因此出现了许多新技术被程序员们开发出来：

1. 从zip压缩包中读取，目前发展为从jar、ear、war包中读取
2. 从网络中获取，主要应用为Web Applet
3. 运行时计算生成，常用于动态代理
4. 由其它文件生成，典型场景为jsp，由jsp文件生成对应的Class文件
5. 从数据库读取
6. 从加密文件中获取

通过重写类加载器的**findClass()**和**loadClass()**方法，可以按自己想要的方式查找代表这个类的二进制字节流和将这个字节流代表的静态存储结构转化为运行时存储结构

#### 2. 什么叫非数组类型？什么叫数组类型？它们都是class吗？

* 实例对象的对象头中的数组长度字段如果被设置，则为数组类型的实例
* 数组类型并不是由类加载器加载进内存的，而是由JVM计算动态构造的
* 他们在方法区均有class对象，但是数组类型并没有静态二进制字节流
* 数组类型的Loading需要递归Loading
* 可以使用对象的**getClass()**方法获取该实例对象的Class对象，调用Class对象中的native **isArray()**方法打印出该对象是不是数组类型
* 当使用System.out.println(getClass())会先调用**getClass()**获取对象，再调用Class的**toString()**方法打印出该实例对象所对应的类名

为了能看懂类名，需要学会看描述符，因为类名是用描述符来显示的

| 标识字符 |      含义       |
| :------: | :-------------: |
|    B     |  基本类型byte   |
|    C     |  基本类型char   |
|    D     | 基本类型double  |
|    F     |  基本类型float  |
|    I     |   基本类型int   |
|    J     |  基本类型long   |
|    S     |  基本类型short  |
|    Z     | 基本类型boolean |
|    V     |  特殊类型void   |
|    L     |    对象类型     |

程序代码

```java
package aboutClass;

public class ComponentTypeVsElementType {
    public static void main(String[] args) {

        Integer notArray=new Integer(10);
        System.out.println("Integer是否是数组类型:"+(notArray.getClass().isArray()?"是":"否"));
        System.out.println("Integer的类型名称:"+notArray.getClass());
        System.out.println("---------------");

        elementCode[] v1=new elementCode[10];
        System.out.println("elementCode[]是否是数组类型:"+(v1.getClass().isArray()?"是":"否"));
        System.out.println("elementCode[]的类型名称:"+v1.getClass());
        System.out.println("elementCode[]的组件类型名称:"+v1.getClass().getComponentType());
        System.out.println("elementCode[]的类加载器:"+v1.getClass().getClassLoader());
        System.out.println("---------------");

        elementCode[][] v2=new elementCode[10][10];
        System.out.println("elementCode[][]的类型名称:"+v2.getClass());
        System.out.println("elementCode[][]的组件类型名称:"+v2.getClass().getComponentType());
        System.out.println("elementCode[][]的类加载器:"+v2.getClass().getClassLoader());
        System.out.println("---------------");

        int[] v5=new int[10];
        System.out.println("int[]的类型名称:"+v5.getClass());
        System.out.println("int[]的类加载器:"+v5.getClass().getClassLoader());
        System.out.println("---------------");

        int[][] v6=new int[10][10];
        System.out.println("int[][]的类型名称:"+v6.getClass());
        System.out.println("int[][]的类加载器:"+v6.getClass().getClassLoader());
    }
}
```

运行结果

```java
Integer是否是数组类型:否
Integer的类型名称:class java.lang.Integer
---------------
elementCode[]是否是数组类型:是
elementCode[]的类型名称:class [LaboutClass.elementCode;
elementCode[]的组件类型名称:class aboutClass.elementCode
elementCode[]的类加载器:sun.misc.Launcher$AppClassLoader@18b4aac2
---------------
elementCode[][]的类型名称:class [[LaboutClass.elementCode;
elementCode[][]的组件类型名称:class [LaboutClass.elementCode;
elementCode[][]的类加载器:sun.misc.Launcher$AppClassLoader@18b4aac2
---------------
int[]的类型名称:class [I
int[]的类加载器:null
---------------
int[][]的类型名称:class [[I
int[][]的类加载器:null
```

所以，数组类型和非数组类型区别也可以说是

* getClass().toString()的返回值，不包括 **[** 的类型为非数组类型
* 数组类型的在堆中生成的实例对象的数组长度字段被设置了
* getClass().isArray()返回值为**true**的类型为数组类型
* 数组类型的组件类型就是数组类型名去掉了一个**[**

#### 3. 书中说非数组类型的Loading对于开发人员来说是可控最强的，那数组类型的Loading有哪里控制不了呢？

数组类型的Loading需要去Loading组件类型，如果组件类型还是数组类型，那仍然要递归Loading组件类型的组件类型，而递归Loading的部分是由JVM实现的。而且数组类型也是由JVM自行动态构造出来的，到最终也是对ElementType的Loading，即对非数组类型的加载，所以说非数组类型的Loading对于开发人员来说是可控最强的，可以重写**findClass()**或**loadClass()**方法来实现类型的动态加载。

* 如果ComponentType是引用类型，则递归Loading，并且把该数组C将被标识在加载该组件类型的类加载器的类名称空间上。
* 如果ComponentType是非引用类型，则将数组类型与引导类加载器绑定
* 数组类型的可访问性与组件类型的可访问性保持一致

#### 4. 引导类加载器是什么？我学过启动类加载器、扩展类加载器、应用程序类加载器、自定义类加载器，和引导类加载器有什么关系？(未解决)

有这个疑问主要是没有动手实现过类加载器，对类加载器不熟悉导致。

ClassLoader指的是实现"通过一个类的全限定名去获取这个类的二进制字节流"的代码，JVM设计团队将这个功能放到了JVM外部进行实现，带来了Loading所需三件事的第一件事的繁荣，使得人们可以按照自己的需要赋予应用程序获取运行代码的动态性。例如从zip、jar、war、ear、网络、数据库、计算、加密文件、jsp等地方获取。

ClassLoader的发展历程：最初为了满足web Applet的需要被设计出来，虽然Web Applet在09年左右已经被主流浏览器淘汰，但是ClassLoader却保存了下来，继续在类层次划分、OSGi、程序热部署、代码加密等领域大放异彩。

在JVM的角度来看，只有两种类加载器：

1. 启动类加载器：由C++实现，是虚拟机的一部分
2. 其他所有类加载器：由Java实现，位于虚拟机的外部，继承自抽象类java.lang.ClassLoader

在开发人员来看，Java为三层类加载器，双亲委派的类加载器架构：

1. 启动类加载器
2. 扩展类加载器
3. 应用程序类加载器

[更多的关于加载器的思考](https://monkeydatabase.github.io/articles/2020-08/ClassLoader)

#### 5. 数组类型既然是由虚拟机直接在内存中构造的，那我们研究它干嘛？

虽然数组类型是由虚拟机构造出来，但是在反射时还是需要用到它，相关的类是**java.lang.reflect.Array**

```word
public final class Array
extends Object
The Array class provides static methods to dynamically create and access Java arrays.
Array permits widening conversions to occur during a get or set operation, but throws an IllegalArgumentException if a narrowing conversion would occur.
```

Array继承自Object类，由final关键字修饰，该类提供用于创建和访问数组的静态方法，在get和set操作时允许扩展，但是当收缩时会抛出**IllegalArgumentException**，而查看源代码后发现构造方法是private导致该类无法实例化，除了从Object类继承来的方法，均为static方法，主要有四类：

* 取值，get、getbyte、getchar......、getdouble
* 赋值，set、setbyte、setchar.......、setdouble
* 创建数组，newInstance
* 获取数组长度，getLength

用例：可以通过一个数组实例对象的**getClass().getComponentType()**获取到组件类型，再通过反射获取某个对象的长度，就可以**newInstance()**一个和该对象一样的存储空间，再通过**System.arraycopy()**将原数组的内容复制到新创建的数组中，通过这样可以做到泛型数组。

#### 6. 组件类型在什么情况下是引用类型？什么情况下不是引用类型？

~~这个问题的关键在于什么是引用，会出这个问题是因为对于引用的理解不到位。~~

```word
虽然问题不是对引用的理解不到位，但是笔记仍然保留：

1.第一次见到引用是在局部变量表，reference类型并不等于对象本身，可能是一个指向对象起始位置的指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置。指针的方式就是物理地址，寻址快速，但是程序如果浮动，所有用到该对象地方的引用都需要改，因为每个指向该对象的引用都存的是真实地址，而不是真实地址存放的位置，这样会大大增加对象浮动的代价；而句柄这一类主要是出于虚拟地址，存储目标真实地址，便于程序浮动，程序中寻址的时候只需要拿着句柄查一下，程序浮动时只需要改一个句柄，其他所有位置都会根据句柄访问到新的位置。Hotspot虚拟机采用了直接地址的方式，即指针的方式。
2.第二次见引用是在JVM内存管理，JDK1.2之前，如果reference类型的数据存储的数值代表另外一块存储空间的起始地址，就称该reference类型的数据中存储的是指代表某块内存、某个对象的引用。但是JDK1.2之后，reference进一步细分，分为Strongly Reference、Soft Reference、Weak Reference、Phantom Reference。
3.第三次遇到引用的疑问是在Class文件结构中许多对常量池的索引，在学到这一部分时，我弄不清楚明明它也指向了一部分数据，为什么不叫引用，而叫索引。
4.第四次遇到引用是当学到了Class Loading的Resolution阶段，它是将常量池中的符号引用转换为直接引用。此处指出常量池中的符号引用指的是CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info、CONSTANT_InterfaceMethodref_info、CONSTANT_NameAndType_info这些常量。符号引用指的是使用一组符号描述所引用的目标，符号可以是各种形式的字面量，只要能够无差错精准的地定位到目标即可，**But** JVM字节码常量池中的符号引用所用的符号因为要跨平台，所以在JVM虚拟机规范中进行了明确定义，要求所有的虚拟机都要满足。Resolution阶段的直接引用既可以是指针、也可以是句柄等形式，即包括了reference类型的实现情况。
```

这个问题的关键在于Java有哪些数据类型，会出这个问题是对于Java面向对象理解的不够深刻。

java中的数据类型分为两类：

1. 基本数据类型(8种)：byte、char、double、float、int、long、short、boolean
2. 引用类型：基本数据类型以外的数据类型，如类、接口、数组、枚举、注解、字符串

#### 7. 组件类型不是引用类型时，数组类型默认为public，那什么情况下数组类型是private？

组件类型不是引用类型，则为基本数据类型，即byte、char、double、float、int、long、short、boolean共八种，他们的数组类型默认为public，默认即在未指定可访问性时设定为public，当有程序员指定的访问修饰符时按照程序员的要求进行设定。

```java
int[] a = new int[10];        //默认为public
private int[] b= new int[10]; //private
```

#### 8. 运行时计算生成常用于动态代理，那么ioc和aop哪个用到了运行时计算生成类呢？

要解决这个问题，首先要明白动态代理是什么，书中给了一个例子讲的是在**java.lang.reflect.Proxy**中用**ProxyGenerator.generateProxyClass()**为特定接口生成形式为"*$Proxy"的代理类的二进制字节流。Proxy是一个实现了Serializable接口、继承自Object类、无参构造方法为private、有参构造为protected、除了继承自Object的方法均为static方法的类。

| 修饰符 |      返回值       |        方法名        | 描述                                                         |
| :----: | :---------------: | :------------------: | ------------------------------------------------------------ |
| static | InvocationHandler | getInvocationHandler | 返回特定代理实例对象的InvocationHandler                      |
| static |     Class<?>      |    getProxyClass     | 根据给定的类加载器和接口集合返回一个代理类的Class对象        |
| static |      boolean      |     isProxyClass     | 在给定的Class是使用getProxyClass或者newProxyInstance方法动态生成的代理类时返回true |
| static |      Object       |   newProxyInstance   | 有三个参数：ClassLoader用于定义动态类，Class<?>[] interfaces给出一系列该动态类需要实现的接口，InvocationHandler分派方法处理，返回一个使用该InvocationHandler进行分派且使用特定类加载器并且实现了给定接口的动态类的实例 |

接下来写动态代理是干什么用的。**代理模式**的目的就是在不直接操作对象的前提下对对象进行访问，以常见的例子进行描述就是VPN，出于安全原因，我们会禁止外网访问内网，但是员工在公司外想要访问数据该怎么办呢，我们采用了VPN服务器，我们既然访问不到真实的目标机器，那我们就将请求发送给VPN服务器，由VPN服务器对目标机器发起访问，再由VPN服务器将结果返回给我们，这个过程中VPN服务器就起到了代理的作用。由此可见，我们常常会遇到要访问目标，但出于某些原因无法直接操作目标，就需要设置一个代理去帮助我们访问，这个过程可能需要与代理之间有额外的约定，所以需要有代理类来存储和执行这些约定。

可以看出getProxyClass和getProxyInstance均是根据ClassLoader和Class<?> [] interfaces动态生成代理类。 

AOP是通过代理实现的，实现面向切面编程，对代码进程功能增强

IOC是将创建对象的控制权从程序员手中转向Spring框架，IOC在创建对象时采用的Dependence Injection依赖注入。依赖注入主要是通过setter注入和构造器注入，产生一个类Class的Bean，而DI主要是通过动态代理和反射技术来实现的。

因此IOC和AOP中均大范围使用了动态代理。

[更多的关于代理的思考](https://monkeydatabase.github.io/articles/2020-08/Proxy)

#### 9. servlet和applet有什么区别？

applet是采用Java编写的小应用程序，可以包含在HTML代码中，为了满足applet的需要，提出了ClassLoader的概念，不过applet目前已经不再使用，主流浏览器也不再支持applet的运行环境。applet每一次修改都需要重新打包签名，手续繁琐，且出于java2的安全性，对applet正常调用的html文件已经不能再使用了。applet是java最早期的项目，必须要java的运行时，之后才出现的flash都已经被淘汰了，而且网上说applet从来都没有流行过。目前浏览器原生支持h5，简单便捷。而且applet是前端插件，本不属于后端的内容，但是由于前端又不会写这些，对于分工协作也不友好。

servlet是Server Applet的简称，是Java编写的服务器端程序。工作流程如下：

1. 客户端对Web服务器发出请求
2. Web服务器接收到请求后将其发送给Servlet
3. Servlet容器为此产生一个实例对象，调用ServletAPI中的方法对请求进行处理
4. Servlet构建一个响应，返回给Web服务器
5. Web服务器将响应返回给用户

相似之处：

1. 都不是独立的应用程序，没有mian方法
2. 都不是由用户或程序员调用，而是由其他程序调用
3. 都有一个生存周期，有**init()**和**destroy()**方法

不同之处：

1. Applet有AWT图形界面，与浏览器一起，在客户端运行
2. Servlet没有图形界面，运行在服务器

#### 10. Integer和int的关系？int没有getClass方法，Integer运行getClass打印出的名字是class java.lang.Integer而不是I，为什么？

1. int是基础数据类型；Integer是int的包装类。int到Integer叫装箱，Integer到int叫拆箱
2. Integer必须实例化才能用；int则不用初始化，因为int是基础数据类型，有直接操作和存储基础类型的字节码指令
3. Integer实际上是对象的引用，是reference类型，**new Integer()**实际上是创建一个Integer并将首地址赋值给Integer这个引用；int则是直接的数据指
4. Integer默认值是Null；int默认值是0

**getClass()**实际上是类的方法，自然只有类才能有。至于打印出来的类名是class java.lang.Integer，是因为它reference指向的对象就是一个Integer实例，而不是实际的数量值。无论什么时候，**getClass()**打印都不会只打印出**I**，最接近打印出来的是**[I**,这是int类型的数组类型。

#### 11. 为什么明明检查类型名的String的第一个字符是否是**[** 就能判断是否是数组类型，那为什么还要用一个native方法去检查是否是数组类型？(未解决)

未能从OpenJDK中定位到**isArray()**的代码，将来需要手动实现一个native方法，进一步了解

#### 12. final关键字有什么用？

1. 类访问标志，ACC_FINAL 0x0010，标志类声明为不能被继承
2. 字段访问标志，ACC_FINAL 0x0010，标志字段不可变，即为常量，不可以与ACC_VOLATILE同用
3. 方法访问标志，ACC_FINAL 0x0010，标志方法不可被重写

#### 13. 既然除了八种基础数据类型以外，均为引用类型，那么Loading的递归出口在哪里？

Loading分为三件事，通过全限定名获取二进制字节流、将二进制字节流代表的静态存储结构转化为运行时数据结构，在堆区创建Class对象作为访问入口。

之所以对这个有疑问，是对最后一个递归理解不清。如果是引用类型，则递归自然没错啦，假设有个类叫Test_v1，Test_v1[10] [10] [10]:

1. 第一次递归组件类型为[[Test_v1，为引用类型进入第二层
2. 第二层递归组件类型为[Test_v1，为引用类型进入第三层
3. 第三层递归组件类型为Test_v1，为引用类型进入第三层。Loading Test_v1时，根据它定位到了二进制字节流，则递归返回
4. .....依次返回

因此它是有递归出口的，递归的深度等同于数组的维度。

每个类型与类加载器一起确定唯一性，在不自定义ClassLoader的情况下，ClassLoader由三层类加载器、双亲委派机制的类加载器架构完成。

#### 14. 基础数据类型、对象、数组有什么区别？

JOI是Java Object Layout，用于打印堆中的对象数据。

接下来讲解它们存储的差异：

* 基本数据类型直接存储在对象的实例数据区、虚拟机栈的操作数栈和局部变量表，可由字节码指令直接操作和运算
* 对象存储在堆区，包括对象头、实例数据、对其填充。执行程序时，通过虚拟机栈帧中局部变量表的reference变量进行寻址。
* 数组也是对象，只不过对象头中多了四个字节用于存储对象长度，且数组类型与组件类型的类型指针指向的位置不同，只是通过jol打印时实例数据区没法解析出来，可能是由于类文件结构的限制，不过存储空间大小是正确的，引用类型占4B，基本类型仍是按实际大小存储。

#### 15. 书中说类加载机制在很多领域有作用，这些领域中的的类层次划分、OSGi指的是什么？

OSGi 面向服务网关协议，是动态模块化的一种规范。

类层次划分，是指不同的层次有不同的作用范围，用于分层和共享。可以以Tomcat为例，它加载类的顺序：

1. jre的java基础包
2. web应用web-inf\lib下的包
3. tomcat\lib下的包

## 产生过的疑问

1. 如何通过全限定名获取定义此类的二进制字节流？
2. 什么叫非数组类型？什么叫数组类型？它们都是class吗？
3. 书中说非数组类型的Loading对于开发人员来说是Proxy可控最强的，那数组类型的Loading有哪里控制不了呢？
4. 引导类加载器是什么？我学过启动类加载器、扩展类加载器、应用程序类加载器、自定义类加载器，和引导类加载器有什么关系？
5. 数组类型既然是由虚拟机直接在内存中构造的，那我们研究它干嘛？
6. 组件类型在什么情况下是引用类型？什么情况下不是引用类型？
7. 组件类型不是引用类型时，数组类型默认为public，那什么情况下数组类型是private？
9. 运行时计算生成常用于动态代理，那么ioc和aop哪个用到了运行时计算生成类呢？
10. servlet和applet有什么区别？
11. Integer和int的关系？int没有getClass方法，Integer运行getClass打印出的名字是class java.lang.Integer而不是I，为什么？
12. 为什么明明检查类型名的String的第一个字符是否是**[** 就能判断是否是数组类型，那为什么还要用一个native方法去检查是否是数组类型？
12. final关键字有什么用？
14. 既然除了八种基础数据类型以外，均为引用类型，那么Loading的递归出口在哪里？
15. 基础数据类型、对象、数组有什么区别？
15. 书中说类加载机制在很多领域有作用，这些领域中的的类层次划分、OSGi指的是什么？