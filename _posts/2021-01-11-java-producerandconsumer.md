---
layout: post
title: 同步互斥调度-生产者消费者
excerpt: "本文将介绍使用Java实现生产者消费者问题。"
date:   2021-01-11 09:27:00
categories: [Java]
comments: true
---

## 学习笔记

### 1、synchronized

#### 1.1 临界区类

```java
import java.util.ArrayList;
import java.util.List;

public class Storage {
    public static final int MAX_SIZE=10;
    private List<Object> list=new ArrayList<Object>();
    
    public void produce(){
        synchronized (list){
            while(list.size()+1>MAX_SIZE){
                try{
                    list.wait();
                }
                catch (InterruptedException e){
                    e.printStackTrace();
                }
            }
            list.add(new Object());
            System.out.println("生产者"+Thread.currentThread()+"生产了一件产品，目前库存为"+list.size());
            list.notify();
        }
    }
    
    public Object consume(){
        Object o=null;
        synchronized (list){
            while(list.size()<1){
                try{
                    list.wait();
                }
                catch (InterruptedException e){
                    e.printStackTrace();
                }
            }
            o=list.remove(list.size()-1);
            System.out.println("消费者"+Thread.currentThread()+"消费了一个产品，目前库存为"+list.size());
            list.notify();
        }
        return o;
    }
}
```

#### 1.2 测试类

```java
public class StorageTest {


    public static void main(String[] args) {
        final Storage storage=new Storage();
        Runnable consumer =new Runnable() {
            public void run() {
                storage.consume();
            }
        };

        Runnable producer=new Runnable() {
            public void run() {
                storage.produce();
            }
        };

        for(int i=0;i<5;i++){
            new Thread(consumer).start();
        }

        for(int i=0;i<10;i++){
            new Thread(producer).start();
        }
    }
}
```

### 2、ReentrantLock

#### 2.1 临界区类

```java
import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Storage {
    private final int MAX_SIZE=10;
    private LinkedList<Object> list=new LinkedList<Object>();
    private Lock lock=new ReentrantLock();
    private Condition condition=lock.newCondition();

    public void produce(){
        lock.lock();
        try{
            while(list.size()==MAX_SIZE){
                System.out.println("生产者"+Thread.currentThread().getName()+"发现库存已满");
                condition.await();
            }
            list.add(new Object());
            System.out.println("生产者"+Thread.currentThread().getName()+"生产了一个产品，当前库存为"+list.size());
            condition.signalAll();
        }
        catch (InterruptedException e) {
            e.printStackTrace();
        }
        finally {
            lock.unlock();
        }
    }

    public void consume(){
        lock.lock();
        try{
            while(list.size()==0){
                System.out.println("消费者"+Thread.currentThread().getName()+"发现库存已空");
                condition.await();
            }
            list.remove();
            System.out.println("消费者"+Thread.currentThread().getName()+"消费了一个产品，当前库存为"+list.size());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}
```

#### 2.2 测试类

```java
public class StorageTest {
    public static void main(String[] args) {
        final Storage storage=new Storage();
        Runnable consumer =new Runnable() {
            public void run() {
                storage.consume();
            }
        };

        Runnable producer=new Runnable() {
            public void run() {
                storage.produce();
            }
        };

        for(int i=0;i<5;i++){
            new Thread(consumer).start();
        }

        for(int i=0;i<7;i++){
            new Thread(producer).start();
        }
    }
}
```



