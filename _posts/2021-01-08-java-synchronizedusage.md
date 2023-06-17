---
layout: post
title: synchronized关键字使用场景
excerpt: "synchronized关键字是系统自动获取和释放，那么使用过程中有哪些应用场景，这些场景中又是获取了什么对象的锁？"
date:   2021-01-08 14:27:00
categories: [Java]
comments: true
---

## 学习笔记

>  Java中的每个对象都可以作为锁。

1. 普通同步方法，锁当前实例对象

   ```java
   //HashTable源码的put方法
   public synchronized V put(K key, V value) {
           // Make sure the value is not null
           if (value == null) {
               throw new NullPointerException();
           }
   
           // Makes sure the key is not already in the hashtable.
           Entry<?,?> tab[] = table;
           int hash = key.hashCode();
           int index = (hash & 0x7FFFFFFF) % tab.length;
           @SuppressWarnings("unchecked")
           Entry<K,V> entry = (Entry<K,V>)tab[index];
           for(; entry != null ; entry = entry.next) {
               if ((entry.hash == hash) && entry.key.equals(key)) {
                   V old = entry.value;
                   entry.value = value;
                   return old;
               }
           }
   
           addEntry(hash, key, value, index);
           return null;
       }
   ```

2. 静态同步方法，锁当前类的Class对象

   ```java
   //加锁类
   public class SynchronizedSource {
       public static synchronized void method1(){
           long start = System.currentTimeMillis();
           try {
               Thread.sleep(1000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println("方法1启动时间："+start);
       }
   
       public static synchronized void method2(){
           long start = System.currentTimeMillis();
           try {
               Thread.sleep(2000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println("方法2启动时间："+start);
       }
   
       public synchronized void method3(){
           long start = System.currentTimeMillis();
           try {
               Thread.sleep(3000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println("方法3启动时间："+start);
   
       }
   }
   
   //测试类
   public class test {
       public static SynchronizedSource syntest=new SynchronizedSource();
   
       public static void main(String[] args) {
   
   
           new Thread(new Runnable() {
               public void run() {
                   syntest.method1();
               }
           }).start();
   
           new Thread(new Runnable() {
               public void run() {
                   syntest.method2();
               }
           }).start();
   
           new Thread(new Runnable(){
               public void run() {
                   syntest.method3();
               }
           }).start();
       }
   }
   
   /*-----------------------------------
   方法1启动时间：1610088821924
   方法3启动时间：1610088821925
   方法2启动时间：1610088822924
   -----------------------------------*/
   ```

   * 方法一和方法二对静态方法加锁，锁的是方法区中的Class对象，所以这两个方式抢占临界区，由于方法二抢占失败，只能等方法一执行结束，方法一执行过程中sleep了1000ms，sleep并不会释放锁，因此方法二的启动时间晚于方法一1000ms
   * 方法三是对普通方法加锁，锁的是堆中实例对象，所以启动时间和方法1相差不大

3. 同步代码块，锁的是括号中给出的对象

   ```java
   public class Singleton{
       
       private Singleton(){}
       
       private Singleton instance;
       
       public Singleton getInstance(){
           if(instance==null){
               synchronized(Singleton.class){
                   if(instance==null)
                       instance=new Singleton();
               }
           }
           return instance;
       }
   }
   ```

   