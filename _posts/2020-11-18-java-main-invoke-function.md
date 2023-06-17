---
layout: post
title: Java Main方法无法调用本类中其它方法
excerpt: "Java基础问题"
date:   2020-11-18 17:01:00
categories: [JVM]
comments: true
---

## 问题原因

Java Main方法是一个Java应用程序的入口

1. public，作为启动类，不应只在类的内部访问，而应该是可以从外部访问
2. static，static修饰的方法在方法区中，JVM可以直接访问该方法，而无需通过实例访问
3. void，Java不需要向操作系统返回退出信息，正常退出为0

```java
public class PizzaStore {
    public static void main(String[] args) {
        String loc=""; 
    }
}
```

调用方法的途径有两种：

1. static方法，直接通过*ClassName.FunctionName()*即可调用
2. static方法调用普通方法，通过一个实例化后的对象，*ObjectName.FunctionName()*来调用
3. 类内普通方法互相调用，可以省略ObjectName，直接通过*FunctionName()*来调用，这是因为它们都是通过对象调用，只不过默认调用者和被调用者同属于一个对象中，所以可以省略ObjectName。

## 解决方法

1. 将被调用方法设置为static方法，就可以通过类名.方法名调用

   ```java
   public class PizzaStore {
       public static void main(String[] args) {
           String loc=""; 
       }
       
       static Stirng getLocation(){
           return "beijing";
       }
   }
   ```

2. 在静态方法中new一个该类的对象，通过对象间接调用

   ```java
   public class PizzaStore {
       public static void main(String[] args) {
           PizzaStore pizzastore=new PizzaStore();
           String loc=pizzastore.getLocation; 
       }
       
       Stirng getLocation(){
           return "beijing";
       }
   }
   ```


