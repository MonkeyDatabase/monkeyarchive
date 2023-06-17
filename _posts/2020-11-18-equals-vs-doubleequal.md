---
layout: post
title: equals方法 vs "=="运算符
excerpt: "该疑问来自于我在编码时的一次运行错误，一开始看所有的代码看不出来错误，最终发现是因它而起"
date:   2020-11-18 17:20:00
categories: [JVM]
comments: true
---

## 问题代码

### 代码一

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

public class vs {
    public static void main(String[] args) throws IOException {
        String bbb="aaa";
        System.out.println(bbb=="aaa");	//true
        String str=new vs().getaaa();	
        System.out.println(str=="aaa");	//false
        System.out.println(str.equals("aaa"));	//true
    }
    //通过方法从命令行输入aaa
    public String getaaa() throws IOException {
        String str="";
        BufferedReader strin=new BufferedReader(new InputStreamReader(System.in));
        str=strin.readLine();
        return str;
    }
}
```

### 代码二

```java
public class vs {
    public static void main(String[] args){
        String s1,s2,s3="aaa",s4="aaa";
        s1=new String("aaa");
        s2=new String("aaa");
        System.out.println("s1==s2:"+(s1==s2));	//false
        System.out.println("s1==s3:"+(s1==s3));	//false
        System.out.println("s3==s4:"+(s3==s4));	//true

        System.out.println("s2.equals(s3):"+s2.equals(s3));//true
    }
}
```

## 问题成因

### equals

equals是java.lang.Object类的方法

1. 以下为Object类的equals方法源码

   ```java
   public boolean equals(Object obj) {
           return (this == obj);
       }
   ```

   * 参数为Object对象
   * 使用==操作符进行判断相等

2. 以下为String类的equals方法的源码

   ```java
   public boolean equals(Object anObject) {
           if (this == anObject) {
               return true;
           }
           if (anObject instanceof String) {
               String anotherString = (String)anObject;
               int n = value.length;
               if (n == anotherString.value.length) {
                   char v1[] = value;
                   char v2[] = anotherString.value;
                   int i = 0;
                   while (n-- != 0) {
                       if (v1[i] != v2[i])
                           return false;
                       i++;
                   }
                   return true;
               }
           }
           return false;
       }
   ```

   * 传入参数为Object对象
   * 首先使用==操作符进行判断，如果结果为true，则直接返回true，不再进行多余判断
   * 如果传入的对象是String类型，将Object对象强制转换为String对象
     * 判断两者长度，长度不同直接返回false
     * 将两个String的char\[\]逐字符进行比较，如果发现不同，则返回false
     * 若两个String的char\[\]内容完全一致，则返回true

### ==

==操作符比较两个变量**本身的值**，当值为Reference类型时，不会去比较Reference所指向对象的内容，而是直接比较两个Reference的值，而它们内存单元中所存储的数据为内存地址，所以当==运算符两侧为Refesrence时，实际比较的是内存地址。

### Tips

两例中均有初始赋值"aaa"的String和一个"aaa"，它们使用==运算符结果为true，是因为它们都是字符串赋值，在Javac编译时，它们两个均指向了常量池中同一个索引，当类被加载到方法区中后，这两个对象所指向的地址也会相同。