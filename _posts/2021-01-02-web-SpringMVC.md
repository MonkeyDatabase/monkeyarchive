---
layout: post
title: SpringMVC
excerpt: "之前多次看过SpringMVC的流程，但都没有详细记录，本文将记录SpringMVC的流程"
date:   2021-01-02 19:57:00
categories: [WEB]
comments: true
---

## 学习笔记

### SpringMVC执行流程

* DispatchServlet
  * 继承了FrameworkServlet抽象类
  * 是HTTP请求Handler或者Controller的中心化调度程序
* HandlerMapping
  * 是一个接口，内部声明了一个getHandler()方法，其返回值为HandlerExecutionChain
  * 实现该接口的对象持有一些映射(request->Handler)
* HandlerExecutionChain
  * 是一个具体类，不继承任何类也不实现任何接口
  * 内部有一个HandlerInterceptor[]数组和List\<HandlerInterceptor\>集合，采用了责任链模式
  * 使用supports()方法按链传递，最终返回一个HandlerAdapter

* HandlerAdapter
  * 是一个接口，其内部有三个方法，supports()、handle()、getLastModified()方法
  * 采用了对象适配器模式，handle方法接收一个Controller类的对象，进而将交由该对象进行处理
* ViewResolver
  * 是一个接口

![SpringMVC执行流程图](https://segmentfault.com/img/remote/1460000024416086)

1. 用户发起请求，请求某个URL

2. 前端控制器DispatchServlet接收请求，其doService()内部调用其doDispatch()方法，doDispatch()内部调用getHandler()。getHandler()遍历DispatchServlet内部的List\<HandlerMapping\>请求调用每个HandlerMapping的getHandler方法，如果返回值不为空，则返回类型为HandlerExecutionChain的值，这个值其实就是我们写注解`@RequestMapping`的方法

   > 后端代码中写了很多的Controller类，类内部写了很多`@RequestMapping`注解，一个`@RequestMapping`相当于一个HandlerMapping

3. HandlerMapping返回一个HandlerExecutionChain调用链给前端控制器DispatchServlet，DispatchServlet遍历自己内部的List\<HandlerAdapter\>得到其中supports(查找到的方法)的处理器适配器HandlerAdapter

4. DispatchServlet请求HandlerAdapter进行处理handle，HandlerAdapter调用执行Controller具体内部方法

5. Controller处理完成后，返回modelAndView给HandlerAdapter

6. HandlerAdapter返回modelAndView给DispatchServlet

7. DispatchServlet将modelAndView交给视图解析器ViewResolver解析视图信息

8. ViewResolver将解析后的View返回给DispatchServlet

9. DispatchServlet将View交给jstl解析View对象

10. jst返回解析后的页面HTML代码

11. DispatchServlet返回页面HTML代码给用户