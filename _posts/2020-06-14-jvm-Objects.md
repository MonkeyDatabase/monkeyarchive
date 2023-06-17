---
layout: post
title: 对象
excerpt: "Java中一切皆对象，那么优化程序运行效率一定要熟练掌握对象大小的计算。"
date:   2020-06-14 10:00:00
categories: [JVM]
comments: true
---

## 指针压缩

现在JDK运行在64位机器上，内存地址理论上有64位，其中有16位是保留位，所以64位机器目前实际最大内存为2^48=256TB。当不开启指针压缩时，指针即地址位为64bit=8B。而开启指针压缩，指针地址为32bit=4B。

因为Java中内存对齐是按8B对齐的，所以开启指针压缩时可用的内存最大为2^32*8B=32G

1. 节省空间
2. 提升jvm运行效率

## 空对象

1. 没有任何普通属性的类生成的对象。
2. 不能说没有方法，因为方法存储在方法区的class对象中，不存储在堆的实例对象中
3. 空对象开启指针压缩和不开启指针压缩占用空间均是16B

| 区域         | 名称      | 作用                                        |
|:------------:|:---------:|:------------------------------------------------------|
| 对象头区域   | Mark Word | 并发编程，内部有锁标志                                 |
|      对象头区域   | Klass Pointer | 对象所属的Class对象的内存地址                          |
| 实例数据区   | 实例数据  | 普通属性，在生成对象后由默认构造方法完成赋值，存在堆区 |
| 对其填充区域 | Padding   | JVM中所有的对齐均是按8B对齐                            |

此时jol包可以用来打印对象头，[Maven仓库](mvnrepository.com/search?jol)中可以看到Java Object Layout:Core

```xml
<!-- https://mvnrepository.com/artifact/org.openjdk.jol/jol-core -->
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.11</version>
</dependency>
```

```java
public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(???).toPrintable());//？？？替换为要打印的对象，toPrintable()方法是以表格形式打印
    }
```

### 证明空对象大小

#### 1、开启指针压缩且有数组对象(-XX:+UseCompressedOops)

指针压缩是默认开启的

```
sync.MyLock object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
      0     4        (object header)           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)           47 c1 00 20 (01000111 11000001 00000000 00100000) (536920391)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

#### 2、关闭指针压缩且有数组对象(-XX:-UseCompressedOops)

```
sync.MyLock object internals:
 OFFSET  SIZE   TYPE DESCRIPTION              VALUE
      0     4        (object header)          01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)          00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)          18 36 4d 18 (00011000 00110110 01001101 00011000) (407713304)
     12     4        (object header)          00 00 00 00 (00000000 00000000 00000000 00000000) (0)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

## 内存优化

内存优化一般是调整堆空间，目的是不频繁执行Full GC，就要控制老年代大小，老年代由三部分组成分别是大对象、GC年龄大于15的对象、空间分配担保。其中可以优化的主要是空间分配担保，计算对象大小，避免新生代担保到老年代。

