---
layout: post
title: Web-Java-反序列化漏洞-ysoserial-Vaadin1
excerpt: "Vaadin是一个集成了JavaScript Web组件的Java UI框架，基于该框架，开发者可以使用Java语言开发出高质量的用户界面。本例将介绍ysoserial工程中关于Vaadin框架的一个反序列化漏洞，之所以在Click1利用链之后紧接着介绍Vaadin1利用链是因为它们最终都用到了Xalan的TemplatesImpl.getOutputProperties()方法来实现远程命令执行。"
date:   2021-08-08 12:47:00
categories: [CTF]
comments: true

---

## 背景知识

1. *ysoserial*包含了许多针对通用Java依赖包的小工具链，可以在正常情况下通过使目标程序进行不安全的反序列化实现攻击利用。

2. Gadgets

   ```java
   /*
    * 	BadAttributeValueExpException.readObject()
    *		PropertysetItem.readObject()
    * 			NestedMethodProperty.readObject()
    *				NestedMethodProperty.initialize()
    *					MethodProperty.initGetterMethod(simplePropertyName, propertyClass);
    *		PropertysetItem.toString()
    *			PropertysetItem.getItemProperty()
    *				NestedMethodProperty.getValue()
    *					Method.invoke()
    *						TemplatesImpl.getOutputProperties()
    *							TemplateImpl.newTransformer()
    *   								TemplateImpl.getTransletInstance()
    *   									TemplateImpl.defineTransletClasses()
    *                                      Class.newInstance() /* Invoke malicious constructor of malicious class*/
    */
   ```

3. Java序列化

   * Java原生序列化和反序列化将属性的类名同时写入到了序列化流中
   * Java反序列化时会调用ObjectInputStream的readObject()方法来进行反序列化
   * 无论反序列化是否成功，只要读取到类名就会加载这个类，如果该类有自定义的readObjec()就执行自定义的readObject()方法，如果没有就按照默认的readObject()流程执行，因此只要readObject()存在调用点，无论业务代码中是否使用了这些类，都会执行readObject()过程从而执行恶意代码

4. Vaadin

   * Vaadin是一个集成了JavaScript Web组件的Java UI框架，基于该框架，开发者可以使用Java语言开发出高质量的用户界面。

## 验证过程

* 配置环境

  * `com.vaadin`:`vaadin-server:7.7.14`
  * `com.vaadin`:`vaadin-shared:7.7.14`
  * `xalan`:`xalan:2.7.2`
  * jdk1.8(非必须)

* 修改ysoserial的PayloadRunner，因为它默认的攻击命令是`clack.exe`，但Ubuntu是没有这条指令的，因此需要对`getFirstExistingFile()`进行修改，在Ubuntu系统中因为存在gnome-calculator，所以如果攻击成功会启动Ubuntu的计算器

  <img src="https://monkeydatabase.github.io/img/image-20210806105905436.png" alt="image-20210806105905436" style="zoom:150%;" />

  ```java
  private static String getDefaultTestCmd() {
      return getFirstExistingFile(
          "C:\\Windows\\System32\\calc.exe",
          "/Applications/Calculator.app/Contents/MacOS/Calculator",
          "/usr/bin/gnome-calculator",
          "/usr/bin/kcalc"
      );
  }
  
  private static String getFirstExistingFile(String ... files) {
      //        return "calc.exe";
      for (String path : files) {
          if (new File(path).exists()) {
              return path;
          }
      }
      throw new UnsupportedOperationException("no known test executable");
  }
  ```

* 在IDEA中启动Click.java

  * 运行之前
  
    ![image-20210808130524282](https://monkeydatabase.github.io/img/image-20210808130524282.png)
  
  * 运行之后
  
    ![image-20210808130621595](https://monkeydatabase.github.io/img/image-20210808130621595.png)


## 攻击原理

1. 首先在生成Payload后打一个断点，来看一下真实的Payload长什么样子

   ![image-20210808132208956](https://monkeydatabase.github.io/img/image-20210808132208956.png)

   

2. 由于最终序列化对象的最外层是`javax.management.BadAttributeValueExpException`(该类位于rt.jar下，因此99.9%可以反序列化出该类)，首先看`BadAttributeValueExpException.readObject()`

   * 首先从中反序列化出所有的字段放到GetField中

     * 在反序列化字段的过程中一定会调用到val属性的反序列化方法，由于val属性为`PropertysetItem`类型并未自定义readObject方法

     * 接下来进入`PropertysetItem`中存放的元素的反序列化方法，此时元素类型为`NestedMethodProperty`，它实现了自定义`readObject`方法，因此看`NestedMethodProperty.readObject()`

       * 首先调用默认的反序列化方法反序列化出自身
       * 然后获取instance属性的Class对象和propertyName属性调用`NestedMethodProperty.initialized()`

       ```java
       // NestedMethodProperty.readObject()
       private void readObject(java.io.ObjectInputStream in)
           throws IOException, ClassNotFoundException {
           in.defaultReadObject();
       
           initialize(instance.getClass(), propertyName);
       }
       ```

     * 接下来看`NestedMethodProperty.initialized()`，因为此时instance是`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`，因此此时放进去了`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.getOutputProperties()`方法到`NestedMethodProperty`.`getMethods`数组中

       > 这段代码过长且与攻击调用链关系不大，故没有放到本博客中，若有兴趣可以查看相应源代码

       * 根据传入的beanClass和propertyName获取beanClass中propertyName的getter方法，放入NestedMethodProperty的getMethods属性
       * 根据传入的beanClass和propertyName获取beanClass中propertyName的setter方法，放入NestedMethodProperty的setMethod属性

   * 从GetField中取出BadAttributeValueExpException的val对象

   * 根据对象类型调用不同的方法原val对象转换为字符串对象，由于`System.getSecurityManager()`如果没有特别处理这里将会返回为null，因此会进入该分支，进而调用`valObj.toString()`

   ```java
   //BadAttributeValueExpException.readObject()
   private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
       ObjectInputStream.GetField gf = ois.readFields();
       
   	/* 如果未关闭IDEA调试功能自动调用toString(),将在这一句攻击生效------------*/
       Object valObj = gf.get("val", null);
   	/* 如果未关闭IDEA调试功能自动调用toString(),将在这一句攻击生效------------*/
       
       if (valObj == null) {
           val = null;
       } else if (valObj instanceof String) {
           val= valObj;
       } else if (System.getSecurityManager() == null
                  || valObj instanceof Long
                  || valObj instanceof Integer
                  || valObj instanceof Float
                  || valObj instanceof Double
                  || valObj instanceof Byte
                  || valObj instanceof Short
                  || valObj instanceof Boolean) {
           val = valObj.toString();
       } else { // the serialized object is from a version without JDK-8019292 fix
           val = System.identityHashCode(valObj) + "@" + valObj.getClass().getName();
       }
   }
   ```

3. 因为val值是`PropertysetItem`类型，因此接下来看`PropertysetItem.toString()`

   * 它会遍历当时通过`addItemProperty()`放入的所有propertyId，将`propertyId`传入`getItemProperty()`获取当时放入的含有恶意载荷的`NestedMethodProperty`对象
   * 接下来就是触发漏洞的**关键几步**，它获取到当初放入的`NestedMethodProperty`之后，触发了它的`getValue()`方法

   ```java
   // addItemProperty() put id and property into it
   private HashMap<Object, Property<?>> map = new HashMap<Object, Property<?>>();
   // addItemProperty() put id into it
   private LinkedList<Object> list = new LinkedList<Object>();
   
   public String toString() {
       String retValue = "";
   
       for (final Iterator<?> i = getItemPropertyIds().iterator(); i.hasNext();) {
           final Object propertyId = i.next();
           retValue += getItemProperty(propertyId).getValue();
           if (i.hasNext()) {
               retValue += " ";
           }
       }
       return retValue;
   }
   
   public Property getItemProperty(Object id) {
       return map.get(id);
   }
   ```

4. 接下来看`NestedMethodProperty.getValue()`，它通过反射调用了`NestedMethodProperty`的`instance`属性`getMethods`中的全部方法，这其中就包含之前初始化时放进去的`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.getOutputProperties()`方法，因此出发了该方法

   ```java
   public T getValue() {
       try {
           Object object = instance;
           for (Method m : getMethods) {
               object = m.invoke(object);
               if (object == null) {
                   return null;
               }
           }
           return (T) object;
       } catch (final Throwable e) {
           throw new MethodException(this, e);
       }
   }
   ```

5. 接下来看`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.getOutputProperties()`，其调用过程与[Click1调用链](https://monkeydatabase.github.io/articles/2021-08/Web-Java-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E-ysoserial-Click1)完全一样

   1. `TemplatesImpl.getOutputProperties()`方法内部调用同类内的方法`TemplatesImpl.newTransformer()`

      ```java
      public synchronized Properties getOutputProperties() { 
          try {
              return newTransformer().getOutputProperties();
          }
          catch (TransformerConfigurationException e) {
              return null;
          }
      }
      ```

   2. 接下来看`TemplateImpl.newTransformer()`，它首先调用了`getTransletInstance()`方法获取一个`Translet`，我们构造的command实际上被封装在了一个Translet类内，因此这是实现攻击**关键的最后几步**。

      ```java
      public synchronized Transformer newTransformer()
          throws TransformerConfigurationException 
      {
          TransformerImpl transformer;
      
          transformer = new TransformerImpl(getTransletInstance(), _outputProperties, _indentNumber, _tfactory);
      
          /* Leave out lots of code */
          return transformer;
      }
      ```

   3. 接下来看`TemplateImpl.getTransletInstance()`

      * 检查`TemplateImpl._name`属性是否为空，如果为空就返回为空，获取`Translet`失败，因此如果使恶意载荷`Translet`顺利被执行，`TemplateImpl._name`必须不为空
      * 检查`TemplateImpl._bytecodes`属性是否为空，看源码中对该属性的注释可知这个`byte[][]`类型的值内存储着`translet`类和一些辅助类的字节码
      * 由源码的注释可知，`TemplateImpl._class`中存储了所有从`TemplateImpl._bytecodes`中通过`TemplateImpl.defineTransletClasses()`方法解析出来的Class对象
      * `TemplateImpl._transletIndex`存储了`TemplateImpl.defineTransletClasses()`在解析字节码过程中发现的`org.apache.xalan.xsltc.runtime.AbstractTranslet`的子类在`TemplateImpl._class`中的地址
      * `AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance()`通过反射调用了我们构造的Template中包含恶意指令的`Translet`的恶意构造方法，**攻击成功**


## 独立思考

### 1、为什么在IDEA中调试Vaadin1调用链时漏洞会被提前触发？

* 在调试Vaadin1调用链时，我很长时间以为攻击成功点位于`BadAttributeValueExpException.readObject()`的如下位置，经过对该语句前后的调用代码全部调试之后仍然找不到调用点，甚至gf.get()方法最后一句也不会触发漏洞利用

  ```java
  private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
      ObjectInputStream.GetField gf = ois.readFields();
      // (IDEA导致误判)攻击成功(假)前 -------------------------
      Object valObj = gf.get("val", null);
      // (IDEA导致误判)攻击成功(假)后 -------------------------
      /* Leave out lots of code */
  }
  ```

* 一次操作让我发现只要调试时从IDEA跳转回`BadAttributeValueExpException.readObject()`时，IDEA的Debugger中会显示当前方法内的变量，在不到1s的时间后漏洞会被触发，因此我猜测是IDEA的Debugger显示当前变量时通过反射调用了这些变量的`toString()`方法，恰好`Vaadin1`调用链依赖`toString()`方法触发，由此导致漏洞会被提前触发。

* 因此尝试**关闭IDEA Debugger自动调用`toString()`**功能，再次调试，发现顺利解决漏洞被提前触发的问题。

  ![image-20210808142611769](https://monkeydatabase.github.io/img/image-20210808142611769.png)

## 产生过的疑问

1. 为什么在IDEA中调试Vaadin1调用链时漏洞会被提前触发？