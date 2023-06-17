---
layout: post
title: Servlet
excerpt: "Java Servlet是Java Web的基础，只有理解了Java Servlet才能更好地理解Spring等Web 框架"
date:   2021-01-03 09:16:00
categories: [WEB]
comments: true
---

## 学习笔记

### Servlet包结构

1. servlet-api可以通过Maven快速引入依赖
2. Servlet是一个javax.servlet.Servlet的接口
   * GenericServlet抽象类实现了Servlet接口，service()方法仍为抽象方法
     * HttpServlet抽象类类继承了GenericServlet抽象类，实现了service()，service判断请求类型，决定是调用doGet、doPost、doHead、doOptions、doDelete、doPut、doTrace

> Spring中的DispatchServlet就是通过继承几个抽象类实现了Servlet接口，具体如下：
>
> DispatchServlet->FrameworkServlet->HttpServletBean->HttpServlet->GenericServlet->Servlet

### Servlet和CGI的区别

1. Servlet处于服务器进程中，通过**多线程**方式运行其service方法，一个实例可以服务于多个请求，并且其实例一般不会销毁，而且Servlet在多线程下使用了同步机制，因此它是**线程安全**的；CGI对于每个请求都产生新的**进程**，服务完成后就**销毁**，所以效率低于Servlet
2. Servlet具有Java的**平台无关**性；传统CGI不具有平台无关性，系统环境发生变化CGI程序就要瘫痪，
3. Servlet具有**连接池**的概念，它利用多线程的优点，在系统缓存中实现建立好若干与数据库的连接，当使用时可以直接从数据库连接池中取；传统CGI一般为两层架构，在网站访问量大的时候，无法克服CGI程序与数据库建立连接速度慢的瓶颈。

### Servlet和JSP的关系

1. 在Servlet逐渐流行之后，人们发现为了能够输出HTML格式内容，需要编写大量的**重复代码**，造成不必要的劳动。为了解决这个问题，基于Servlet产生了Java Servlet Pages，即JSP。
2. Servlet和JSP分工协作，Servlet侧重于解决运算和业务逻辑问题，JSP侧重于解决页面展示问题

### Servlet生命周期

* 加载：容器通过类加载器加载Servlet
* 创建：通过构造函数创建一个对象
* 初始化：调用init()初始化Servlet，之后几乎一直驻留内存
* 处理客户端请求：容器每收到一个请求，开启一个线程去处理
* 卸载：destroy()方法仅执行一次，即在服务器停止并卸载Servlet时





