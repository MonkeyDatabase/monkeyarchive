---
layout: post
title: Web进阶题-unserialize3
excerpt: "攻防世界进阶Web题目-unserialize3"
date:   2020-10-11 11:42:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问目标网站，网页正文：

   ```php
   class xctf{
   public $flag = '111';
   public function __wakeup(){
   exit('bad requests');
   }
   ?code=
   ```

2. 看到网页内容，猜测是通过GET方法传一个code参数过去，实现注入

3. 当访问*ip:port/?code=xxx*，网页内容为"?code=xxx"，没有有效输出

4. 题目名称为反序列化，则学习一下PHP反序列化相关的漏洞

5. 构造反序列化对象xtcf，网址为*ip:port/?code=O:4:"xctf":1:{s:4:"flag";s:5:"11111";}*，网页内容为”bad request“

6. 目前是反序列化出了xctf对象，但是反序列化unserialize()会先调用魔术方法wakeup，导致PHP脚本执行退出，所以需要绕过该魔术函数.*ip:port/?code=O:4:"xctf":2:{s:4:"flag";s:5:"11111";}*

7. 返回flag，提交答案正确。

## 独立思考

### 1. PHP反序列化是如何做的？

* 以*；*作为字段的分割
* 以}作为结尾
* 根据长度判断内容
* 整体对象序列化后为*type:elementCount:{}*，花括号内存放序列化后的元素。
  * 当为对象时，type为*O:classNameLength:className*
  * 当为数组时，type为*a*
* 元素或属性
  * 花括号内的元素依次为key1 value1 key2 value2 ......
  * 数值型元素*type:value*，int的type为*i*
  * 字符串型元素*type:length:"text"*，type为*s*

例：

```php
<?php
$str='a:2:{i:0;s:5:"hello";i:2;s:5:"world";}';
var_dump(unserialize($str));
?>
//运行结果
//array(2) {
//  [0]=>
//  string(5) "hello"
//  [2]=>
//  string(5) "world"
//}
```

### 2. PHP中\_\_开头的方法有什么作用？

PHP中保留了所有以“\_\_"开头的方法，这些方法称之为魔术方法。

如果想要PHP在执行过程中自动在调用点调用这些函数，需要在类中进行定义，否则这些未定义的魔术方法不会被执行

| 方法名         | 调用点                                                       |
| -------------- | ------------------------------------------------------------ |
| \_\_set()      | 在写入不可见或未定义的变量时                                 |
| \_\_get()      | 在调用不可见或未定义的变量时                                 |
| \_\_call()     | 在调用不可见或不存在的成员方法时                             |
| \_\_sleep()    | 在调用serialize()函数时，若有sleep魔术方法则先调用魔术方法   |
| \_\_wakeup()   | 在调用unserialize()函数时，若有wakeup魔术方法则先调用魔术方法 |
| \_\_toString() | 当使用echo或print输出对象时，将对象转换为字符串              |
| \_\_autoload() | 当程序使用一个类，但它还没有实例化时                         |

本题中使用了__wakeup()，且该方法体中有exit，所以是没办法反序列化我们输入的内容的。

因此本题需要想办法绕过wakeup()魔术方法。

### 3. 如何绕过PHP的wakeup魔术方法？

* 只需反序列化对象的属性(元素)个数比花括号中的属性(元素)个数多，就可以绕过\_\_wakeup()魔术方法。
* 但是除了属性(元素)个数不一致以外，其他均应保持正确，否则反序列化会失败

## 产生过的疑问

1. PHP反序列化是如何做的？
2. __开头的方法有什么作用？
3. 如何绕过PHP的wakeup魔术方法？

