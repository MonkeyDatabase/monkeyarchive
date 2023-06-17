---
layout: post
title: Web新手题-simple_php
excerpt: "攻防世界新手Web题目-simple_php"
date:   2020-10-10 15:56:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“小宁听说php是最好的语言,于是她简单学习之后写了几行php代码。”

2. 访问网页，返回了php源码

   ```php
   <?php
   show_source(__FILE__);
   include("config.php");
   $a=@$_GET['a'];
   $b=@$_GET['b'];
   if($a==0 and $a){
       echo $flag1;
   }
   if(is_numeric($b)){
       exit();
   }
   if($b>1234){
       echo $flag2;
   }
   ?>
   ```

   * show_source() 函数对文件进行 PHP 语法高亮显示
   * \_\_FILE\_\_取当前文件的绝对地址
   * @在PHP中用作错误控制操作符，当表达式加有@将忽略该表达式可能生成的错误信息
   * 获取了a和b两个GET方法传来的参数
     * a与数值0进行判断值相等，同时a变量被定义，才会打印flag1
     * b需要不是数值而且要数值大于1234，才会打印flag2
     * flag1和flag2应该是在config.php中进行定义的

3. 这道题一定涉及到了强制类型转换，因为b变量的要求前后矛盾

4. 由于之前看到PHP将字符串转为数值是从头开始选取数值直到不是数值为止，所以可以把a设置为“aaaa”，b设置为“12345b”

5. 发起请求，ip:port/?a=aaaaa&b=12345b

6. 发现flag：Cyberpeace{647E37C7627CC3E4019EC69324F66C7C}

7. 提交，答案正确。

## 独立思考

### 1. PHP类型转换有哪些需要注意的点？

| 操作符 | 名称 | 作用                                             |
| ------ | ---- | ------------------------------------------------ |
| ==     | 等于 | 弱类型比较，在比较前会进行自动类型转换           |
| ===    | 全等 | 强类型比较，两个操作数不仅值相等，数据类型也相等 |

强制类型转换有两种方式：

* (类型名)，变量或表达式
* 类型转换函数，intval()、floatval()、strval()、settype()等

PHP将字符串转为数值是从头开始选取数值直到不是数值为止，也正是本题解题的关键所在。

```php
<!DOCTYPE html> 
<html> 
	<body> 
		<?php 
			$a=1;
			echo floatval("123.23fsd5");	//输出123.23，而不是123.235
			echo "\n";
			echo @($a+"a");	//加法运算符 输出1，"a"被自动转为int
			echo "\n";
			echo @($a."a");	//连接运算符 输出1a
		?> 
	</body> 
</html>
```

## 产生过的疑问

1. PHP类型转换有哪些需要注意的点？
