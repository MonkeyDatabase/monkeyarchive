---
layout: post
title: Java Bean
excerpt: "Java Bean->EJB->POJO,PO、VO、DAO、DTO概述"
date:   2020-06-17 13:34:00
categories: [WEB]
comments: true
---



# Java Beans

## Java Beans初衷

Java语言欠缺属性、事件、多重继承功能，在Java中实现一些面向对象编程的常见需求，只能手写大量胶水代码。Java bean其实是一种规范，用于规范胶水代码的书写。

Java bean其实就是一个普通的Java类，但有一些规定：

1. 这个类需要是public的，然后有个无参数的构造函数
   * 在可视化开发方式中，当需要拖动创建一个组件的时候，就需要把这个类通过反射给new出来，所以需要一个无参数的构造函数
2. 这个类的属性应该是private的，通过setXXX()和getXXX()
   * 当用户想要设置这个对象的属性时，可视化工具就可以用自省/反射来获取这个对象有哪些属性，拿到以后可以列一个属性清单，还可以设置对象属性
3. 这个类需要能支持事件，例如addXXXListener(XXXEvent e)，事件可以是Click事件或者Keyboard事件等等，当然也支持自定义的事件
   * 当用户想要自定义功能，可以通过自省/反射来获取这个对象有哪些事件，给用户列一个事件清单，拿到事件清单后，用户可以对事件进一步定义
4. 需要提供一个自省/反射机制，这样能在运行时查看java bean的各种信息
5. 这个类应该是可序列化的，即可以把bean的状态保存在硬盘上，以便以后用来恢复。

最终，Java在桌面开发市场仍是没有起色，之后转向了服务器端编程

## JSP+Java Bean

### JSP Model 1

![img](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6UJUeoI1NHa8pDTH0VGgy4Fu4Zq87VCxLtXs3KYW3KtXlzkNAbyrDtDoBRqI7f3hGJ1BlOMqL5AJwA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

用Java Bean封装业务逻辑，保存数据到数据库

```jsp
<jsp:useBean/>标签可以在JSP中声明一个Java Bean，然后使用。声明后JavaBean对象就成了脚本变量，可以通过脚本元素或其他自定义标签来访问。
<jsp:useBean id="Bean的名字" scope="Bean的作用范围" class="java bean的全限定名"/>
id只需与同一JSP文件下的其它<jsp:useBean id="">重复即可
scope可以为page、session、request、application

而使用java bean即使用property标签来调用java bean的getter和setter方法
<jsp:setProperty name="java bean的名字"  property="属性名" value="属性的值"/
                                        property="属性名" param="将一个请求参数的值赋给javabean的某个属性"/
                 						property="*"/
                 						property="属性名"
                 >
param和value不能同时使用
当缺省param和value时，将自动获取与property同名的请求参数的值并赋给这个属性

当嵌套在<jsp:useBean></jsp:useBean>时，表示只有实例化对象的时候执行一次，用于初始化对象。若对象已存在，则不会执行
<jsp:useBean id="" scope=" " class=" ">
    <jsp:setProperty name=" " property=" " param=" "/>
</jsp:useBean>
    
<jsp:getProperty name="java bean的名字" property="属性名"/>
调用java bean对象的getter方法，将结果转化为字符串后输出到响应正文
```



```jsp
<jsp:useBean id="user" scope="page" class="com.coderising.User"/>
<jsp:setProperty property="userName" name="user" param="userName"/>
<jsp:setProperty property="password" name="user"param="password"/>
```

### JSP Model2

因为java bean负责业务逻辑，jsp负责接收请求的模式，在项目特别大的时候，会出现理不清楚调用关系的问题，所以推出了JSP Model2，真正体现了Model-View-Controller的思想，servlet充当controller、jsp充当view、javabean充当model，将业务逻辑、页面显示、处理过程进行分离。



![img](http://mmbiz.qpic.cn/mmbiz/KyXfCrME6UJUeoI1NHa8pDTH0VGgy4Fu2DzGia9CYhbLDTvkcYCibg3Urfur0zFOaL3cmNBZWX0cXbtoq8cibyTEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## EJB

为了满足企业应用服务的需求，Java推出了Java EE。其中包括了：

1. JDBC: Java数据库连接
2. JNDI: Java命名和目录护额基于
3. RMI: 远程过程调用
4. JMS: Java消息服务
5. JTA: Java事务管理
6. Java Mail: 邮件收发服务
7. EJB: Enterprise Java Bean

## POJO

POJO(Plain Old Java Object) 简单Java对象。

POJO的内在含义是指没有继承任何类、也没有实现任何接口、更没有被其他框架侵入的java对象

### 为什么会有POJO

EJB过于繁琐

### POJO的意义

POJO让开发者更关注业务逻辑和脱离框架的单元测试。

因为POJO的简单灵活，使得POJO能够任意扩展，胜任多个场合。先写一个POJO，然后实现业务逻辑接口和持久化接口就成了Domain Model，UI需要使用时就实现数据绑定接口变成VO

### POJO、VO、PO

![img](https://pic4.zhimg.com/80/v2-bbac0456af84c9feb17b03cdd9501222_720w.jpg)

* PO是指persistence object持久对象
* VO是指value object值对象、或者view object

> 持久化对象实际上必须对应数据库中的entity(与数据库中字段一致)
>
> 1. @Transient  不是数据库字段的属性要加这个注解
> 2. @Column(name=" ") 当数据库字段与result不一致时用这个注解
> 3. @Param 传入参数与数据库字段不一致时用这个注解

**注意**

1. POJO由new创建，由GC回收
2. PO是insert数据库创建、数据库delete删除。PO生命周期和数据库密切相关，而且PO往往只能存在一个数据库Connection之后。Connection关闭之后，PO将不复存在。

### PO

用来表示数据库中一条记录映射成的java对象，仅仅用于表示数据

#### PO状态

![img](https://images2015.cnblogs.com/blog/702434/201510/702434-20151004095413746-2129601422.png)

1. 瞬时态(暂态)：实体在内存中自由存
2. 在，与数据库记录无关。po在db中无记录，与session无关。(手动管理同步)
3. 持久态(Persistent)：实体对象处于Hibernate框架管理之中。po在db中有记录，和session有关。(Session自动管理同步)
4. 游离态(Detached)：处于Persitent状态的实体对象，其对应的Session实例关闭后，此时的实体对象处于Detached。po在db中有记录，与session无关(手动管理同步)

> @Entity 标注在类上，声明为实体类
>
> @Table(name=" ") 指定数据库的表
>
> @Id 指定表的主键
>
> @Column(name=" ") 标注在属性上，声明数据库字段名 

