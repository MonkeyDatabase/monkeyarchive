---
layout: post
title: Web新手题-backup
excerpt: "攻防世界新手Web题目-backup"
date:   2020-10-05 11:44:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述为“X老师忘记删除备份文件，他派小宁同学去把备份文件找出来,一起来帮小宁同学吧”

2. 访问网页，网页内容为“你知道index.php的备份文件名吗？”

3. 使用子目录遍历工具dirbuster，进行遍历，遍历了很长时间都没有出结果

4. 在网上搜索常见的备份文件命名方法，发现是在原文件名后加*.bak*

5. 访问ip:port/index.php.bak，下载到一个文件

6. 使用Hex Editor打开，内容如下：

   ```php+HTML
   <html>
   <head>
       <meta charset="UTF-8">
       <title>澶囦唤鏂囦欢</title>
       <link href="http://libs.baidu.com/bootstrap/3.0.3/css/bootstrap.min.css" rel="stylesheet" />
       <style>
           body{
               margin-left:auto;
               margin-right:auto;
               margin-TOP:200PX;
               width:20em;
           }
       </style>
   </head>
   <body>
   <h3>你知道index.php的备份文件名吗？</h3>
   <?php
   $flag="Cyberpeace{855A1C4B3401294CB6604CCC98BDE334}"
   ?>
   </body>
   </html>
   ```

7. 发现flag：Cyberpeace{855A1C4B3401294CB6604CCC98BDE334}

8. 提交，答案正确

## 独立思考

### 1. 如何才能高效利用子目录爆破工具？

* 本题中我使用了dirbuster，这个工具中可以选用File extension设置文件名后缀，我开始时就给设成了php，所以无论我扫描多久，都不会出来结果。
* 而且由于对于Fuzz时限制条件了解不够，导致扫描所需时间过长，完全不适用于实战。
* 要熟练使用正则表达式减少不必要的发包

### 2. 子目录爆破时有哪些小技巧？

* 常见备份文件后缀名为“.git"、".svn"、".swp"、"~"、".bak"、".bash_history"、".bkf"，通常是在原文件名后添加这些新后缀以便区分

### 3. 使用Hex Editor打开文件后，中文是乱码的，如何解决？

在Hex Editor的Tools->Settings->Editor->Encoding，选为UTF-8，重新打开文件即可。

*e4bda0e79fa5e98193696e6465782e706870e79a84e5a487e4bbbde69687e4bbb6e5908de59097efbc9f*就被显示为了*你知道index.php的备份文件名吗？*

我把这段十六进制复制到了Chrome扩展Hasher中第一次无法解析是因为这段十六进制串的最后多了个回车，导致解码*NaN*。

## 产生过的疑问

1. 如何才能高效利用子目录爆破工具？
2. 子目录爆破时有哪些小技巧？
3. 使用Hex Editor打开文件后，中文是乱码的，如何解决？
