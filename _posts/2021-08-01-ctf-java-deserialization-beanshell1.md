---
layout: post
title: Web-Java-反序列化漏洞-ysoserial-BeanShell1
excerpt: "BeanShell1是由Java语言编写的Java语言解释器，通过Java反射API实现实时解释执行Java语言，类似于Python和JavaScript。BeanShell1正是基于BeanShell组件实现的RCE漏洞，并且反序列过程中并不会有任何报错信息。"
date:   2021-08-01 20:16:00
categories: [CTF]
comments: true
---

## 背景知识
1. ysoserial包含了许多针对通用Java依赖包的小工具链，可以在正常情况下通过使目标程序进行不安全的反序列化实现攻击利用。

2. Gadgets
	
	```JAVA
	/* 
	 *   Gadget chain:
	 *      PriorityQueue.readObject()
	 *          PriorityQueue.heapify()
	 *              PriorityQueue.siftDown()
	 *                  PriorityQueue.siftDownUsingComparator()
	 *                      Comparator.compare()
	 *                      	Proxy0.compare()
	 *                      		XThis$Handler.invoke()
	 *                          			ProcessBuilder.start()
	 */
	```
	
3. Java序列化

  * Java原生序列化和反序列化将属性的类名同时写入到了序列化流中
  * Java反序列化时会调用ObjectInputStream的readObject()方法来进行反序列化
  * 无论反序列化是否成功，只要读取到类名就会加载这个类，如果该类有自定义的readObjec()就执行自定义的readObject()方法，如果没有就按照默认的readObject()流程执行，因此只要readObject()存在调用点，无论业务代码中是否使用了这些类，都会执行readObject()过程从而执行恶意代码

4. BeanShell

   * 用Java语言写成的Java源代码解释器，可以执行标准Java语句和表达式，另外包括一些脚本命令和语法。

   * 使用Java反射API以提供Java语句和表达式的实时解释执行

   * 因为BeanShell是Java语言写成的，可以与其它组件一起运行在同一个JVM中，所以可以透明地访问任何Java对象和API

   * 例如如下代码可以启动本地Firefox浏览器(Linux环境)

     ```java
     public static void main(final String[] args) throws Exception {
         Interpreter i = new Interpreter();
         i.eval("Runtime runtime = Runtime.getRuntime()");
         i.eval("runtime.exec(\"/usr/bin/firefox\")");
     }
     ```


## 验证过程

* 配置环境

  * `org.beanshell`:`bsh:2.0b5`
  * java8(非必须)

* 修改ysoserial的PayloadRunner方法便于展示攻击效果

  * 向默认命令中添加了一行"/usr/bin/firefox"，否则在Linux环境下会因没有有效Command而报错
  * 因为getFirstExistingFile默认是在第一行就返回 calc.exe，但是在Linux环境下没有calc.exe，因此把该句注释掉，紧接下来遍历传入的所有命令用于找出有效的指令

  ```java
  private static String getDefaultTestCmd() {
      return getFirstExistingFile(
          "C:\\Windows\\System32\\calc.exe",
          "/Applications/Calculator.app/Contents/MacOS/Calculator",
          "/usr/bin/gnome-calculator",
          "/usr/bin/kcalc",
          "/usr/bin/firefox"
      );
  }
  
  private static String getFirstExistingFile(String ... files) {
      //return "calc.exe";
      for (String path : files) {
          if (new File(path).exists()) {
              return path;
          }
      }
      throw new UnsupportedOperationException("no known test executable");
  }
  ```

* 攻击效果

  * IDEA终端输出

    ![image-20210801203325557](https://monkeydatabase.github.io/img/image-20210801203325557.png)

  * 反序列化桌面效果，火狐浏览器被启动

    ![image-20210801203503018](https://monkeydatabase.github.io/img/image-20210801203503018.png)

## 攻击原理

1. 通过调试查看一下Payload的具体样子

   ![image-20210809152907522](https://monkeydatabase.github.io/img/image-20210809152907522.png)

2. BeanShell1在构建Payload时，首先通过字符串构造了一段含有恶意指令名称为`compare`的方法源代码，使用bsh中的Interpreter解释执行这段代码，每个`Interpreter`含有一个类型为`NameSpace`的命名空间属性globalNameSpace，其内部会存储当前Interpreter的上下文信息，如所有方法会存储在globalNameSpace的名称为methods的HashTable中，便于后续定位并调用方法。

3. Payload的最外层是一个PriorityQueue，其触发攻击链的逻辑和[Click1](https://monkeydatabase.github.io/articles/2021-08/Web-Java-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E-ysoserial-Click1)类似，PriorityQueue逻辑结构基于堆实现，物理结构基于数组实现

   * `PriorityQueue.readObject()`首先反序列化出PriorityQueue中的全部元素，调用堆化`heapify()`方法

     ```java
     private void readObject(java.io.ObjectInputStream s)
         throws java.io.IOException, ClassNotFoundException {
         s.defaultReadObject();
         s.readInt();
     
         queue = new Object[size];
         for (int i = 0; i < size; i++)
             queue[i] = s.readObject();
         heapify();
     }
     ```

   * `PriorityQueue.heapify()`，要堆化自然就需要比较大小，而此时PriorityQueue中comparator是构造出来的且不为空，因此触发了`comparator.compare()`

     ```java
     private void heapify() {
         for (int i = (size >>> 1) - 1; i >= 0; i--)
             siftDown(i, (E) queue[i]);
     }
     
     private void siftDown(int k, E x) {
         if (comparator != null)
             siftDownUsingComparator(k, x);
         else
             siftDownComparable(k, x);
     }
     
     private void siftDownUsingComparator(int k, E x) {
         int half = size >>> 1;
         while (k < half) {
             int child = (k << 1) + 1;
             Object c = queue[child];
             int right = child + 1;
             if (right < size &&
                 comparator.compare((E) c, (E) queue[right]) > 0)
                 c = queue[child = right];
             if (comparator.compare(x, (E) c) <= 0)
                 break;
             queue[k] = c;
             k = child;
         }
         queue[k] = x;
     }
     ```

4. 因为当前Payload反序列化出来的`Comparator`是`Proxy`实例，因此它会将`comparator.compare()`请求转发给其内部的InvocationHandler来进行处理，此时的InvocationHandler是`bsh.XThis$Handler`类的实例，所以`bsh.XThis$Handler.invoke()`将会从之前创建的命名空间中寻找到之前构造的恶意compare方法进行执行，**攻击成功**

   ```java
   public Object invoke( Object proxy, Method method, Object[] args ) throws Throwable
   {
       try { 
           return invokeImpl( proxy, method, args );
       } 
       /* Leave lots of codes for try-catch */
   }
   ```

## 独立思考

### 1、Proxy和InvocationHandler在动态代理中的作用有哪些？他们是如何配合工作的？

* 首先捕获生成的Proxy，在生成Proxy的语句之前加入如下语句，重新运行程序可以在根目录下找到生成类的Class文件

  ```java
  // Create InvocationHandler
  XThis xt = new XThis(i.getNameSpace(), i);
  InvocationHandler handler = (InvocationHandler) Reflections
      .getField(xt.getClass(), "invocationHandler").get(xt);
  // Create Comparator Proxy
  // 加入下面这一句可以导出生成的Proxy的Class文件 -----------------------
  System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
  // ---------------------------------------------------------------
  Comparator comparator = (Comparator) Proxy.newProxyInstance(Comparator.class.getClassLoader(), new Class<?>[]{Comparator.class}, handler);
  ```

* 打开生成的`Comparator`Proxy类文件，以下为省略相似代码后的内容，如果想了解Proxy的真实样貌请自行调试程序

  * `Proxy0`会继承`Proxy`类从而获取`Proxy`类的一些属性，因为Java只支持单继承且它已经继承了`Proxy`类型，所以Java中的`Proxy`只支持接口继承
  * 由于每个类被虚拟机加载都会加载到一个`ClassLoader`的命名空间中，所以在创建`Proxy0`时需要指定将来加载它的`ClassLoader`，往往这个`ClassLoader`会被设置为加载它所需实现的接口所用的`ClassLoader`，因为不同的`ClassLoader`的命名空间是隔离的
  * `Proxy0`会实现`Proxy.newProxyInstance()`传入的第二个参数`Class<?>[]`接口数组中的全部接口
  * `Proxy0`会接收`Proxy.newProxyInstance()`传入的第三个参数`InvocationHandler`作为生成类$Proxy0构造函数的参数，从而创造出一个`Proxy0`实例
  * `Proxy0`由于实现了规定的接口，所以其Class内部会有各个接口中所定义方法的实现，不过这个实现均如下面代码所示，它将传入的所有参数和被调用的方法全部交给了`InvocationHandler`去进行具体实现，**这也是为什么创建Proxy时必须要用一个InvocationHandler作为构造函数的入参**。

  ```java
  public final class $Proxy0 extends Proxy implements Comparator {
      /* Leave lots of 18 method objects similar to m3 */
      private static Method m3;
  
      public $Proxy0(InvocationHandler var1) throws  {
          super(var1);
      }
  
      /* Leave lots of codes for diffenert methods similar to compare()*/
  
      public final int compare(Object var1, Object var2) throws  {
          try {
              return (Integer)super.h.invoke(this, m3, new Object[]{var1, var2});
          } catch (RuntimeException | Error var4) {
              throw var4;
          } catch (Throwable var5) {
              throw new UndeclaredThrowableException(var5);
          }
      }
  
      static {
          try {
              /* Leave lots of 18 Class.forName() codes similar to m3 */
              m3 = Class.forName("java.util.Comparator")
                      .getMethod("compare", 
                                 Class.forName("java.lang.Object"), 
                                 Class.forName("java.lang.Object"));
          } catch (NoSuchMethodException var2) {
              throw new NoSuchMethodError(var2.getMessage());
          } catch (ClassNotFoundException var3) {
              throw new NoClassDefFoundError(var3.getMessage());
          }
      }
  }
  ```

### 2、为什么`bsh.XThis$Handler`反序列化后其内部有一个`this$0`的属性 ？

* 先看Payload的内部对象具体状况

  ![image-20210809170528662](https://monkeydatabase.github.io/img/image-20210809170528662.png)

* 再看一下Handler的源代码结构

  ![image-20210809170814369](https://monkeydatabase.github.io/img/image-20210809170814369.png)

* 可以看到Handler是XThis的一个内部类，内部类分静态内部类和非静态内部类

  * 静态内部类和外部类没有区别，可以像普通外部类一样使用`new`关键字创建实例。

    ![image-20210809172604658](https://monkeydatabase.github.io/img/image-20210809172604658.png)

  * 非静态内部类只能在包含它的外部类中使用，非静态内部类在创建实例时默认隐含一个指向创建其本身的外部类实例的引用，因此非静态内部类的实例是可以访问其对应的外部类实例的所有属性及方法的。如下图所示，aaa中创建了bbb作为内部属性，bbb默认隐含了一个this$0指向aaa，在IDEA的Debugger里面查看引用的话，就会出现无限套娃的情况。

    ![image-20210809172210677](https://monkeydatabase.github.io/img/image-20210809172210677.png)

* Handler是一个非静态成员类，非静态成员类的每个实例都隐含着与外层类的一个外层类实例，该实例为指向的XThis的引用，所以在序列化时虽然表面只序列化了Handler，实际上连其外部类实例也一同序列化到字节流中了，这样保证了反序列化之前该内部类实例访问到的外部类属性在反序列化后仍能够保持不变。

## 产生过的疑问

1. Proxy和InvocationHandler在动态代理中的作用有哪些？他们是如何配合工作的？
2. 为什么`bsh.XThis$Handler`反序列化后其内部有一个`this$0`的属性 ？

