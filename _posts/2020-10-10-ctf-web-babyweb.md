---
layout: post
title: Web进阶题-babyweb
excerpt: "攻防世界进阶Web题目-babyweb"
date:   2020-10-11 09:41:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“想想初始页面是哪个。”

2. 访问网站，进入了1.php，网页内容只有”HELLO WORLD“，没有注入点

3. 使用Burp Suite爆破文件名，没有反应

4. 使用dirsearch进行子目录爆破，*python3 ./dirsearch.py -u "xxx:xxxx" -e php*

   ```shell
   [20:38:16] 403 -  306B  - /.htaccess.bak1                             
   [20:38:16] 403 -  308B  - /.htaccess.sample
   [20:38:16] 403 -  306B  - /.htaccess.orig
   [20:38:16] 403 -  306B  - /.htaccess.save
   [20:38:16] 403 -  304B  - /.htaccessBAK
   [20:38:16] 403 -  304B  - /.htaccessOLD
   [20:38:16] 403 -  305B  - /.htaccessOLD2
   [20:38:16] 403 -  296B  - /.htm
   [20:38:16] 403 -  297B  - /.html
   [20:38:16] 403 -  303B  - /.httr-oauth
   [20:38:16] 200 -   11B  - /1.php                                                  
   [20:38:21] 302 -   17B  - /index.php  ->  1.php                                                                   
   [20:38:22] 302 -   17B  - /index.php/login/  ->  1.php                                         
   [20:38:24] 403 -  306B  - /server-status/                                                               
   [20:38:24] 403 -  305B  - /server-status
   
   ```

5. 没做出来

## 别人的解法

1. 根据题目描述，访问index.php，发现返回值302，并跳转到了1.php

2. 所以使用Burp Suite抓包，抓取index.php的请求报文，转发到Repeater模块

   ```http
   GET /index.php HTTP/1.1
   Host: 220.249.52.133:45863
   User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:81.0) Gecko/20100101 Firefox/81.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
   Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
   Connection: close
   Upgrade-Insecure-Requests: 1
   Cache-Control: max-age=0
   ```

3. 使用Repeater发包，Response部分显示了响应报文

   ```http
   HTTP/1.1 302 Found
   Date: Sun, 11 Oct 2020 03:14:20 GMT
   Server: Apache/2.4.38 (Debian)
   X-Powered-By: PHP/7.2.21
   FLAG: flag{very_baby_web}
   Location: 1.php
   Content-Length: 17
   Connection: close
   Content-Type: text/html; charset=UTF-8
   
   Flag is hidden!
   ```

4. 可以看到在响应头字段里，有flag，flag{very_baby_web}

## 独立思考

### 1.PHP通过什么实现了自动跳转？

PHP有一个函数*header(string，replace，response_code)*可以设置响应头字段

* string：必需，指定了响应头字段字符串
* replace：可选，当该字段已经在报头有了之后，是替换之前的，还是直接加一条
* response_code：可选，可以强制指定响应的状态码

浏览器在接收到响应码为302时，会自动检查Location响应头，进行跳转。

```php
<?php
    header('FLAG: flag{very_baby_web}');
	header('Location:https://www.baidu.com');
?>
```

所以，通过以上代码，可以直接重定向到百度的页面。

在这个过程中，如果不抓包，就会**看不到**302响应报文的响应头。

## 产生过的疑问

1. PHP通过什么实现了自动跳转？

