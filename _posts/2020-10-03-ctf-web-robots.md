---
layout: post
title: Web新手题-robots
excerpt: "攻防世界新手Web题目-robots"
date:   2020-10-05 11:10:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“X老师上课讲了Robots协议，小宁同学却上课打了瞌睡，赶紧来教教小宁Robots协议是什么吧“

2. 打开网页，发现是一个空白页，浏览源代码只有一个注释，内容是“flag is not here”

3. 根据题目描述，访问robots文件，因为robots.txt是默认存储在网站根目录下的ASCII编码的文本文件，内容如下：

   ```txt
   User-agent: *
   Disallow: 
   Disallow: f1ag_1s_h3re.php
   ```

4. 发现一个文件名为f1ag_1s_h3re.php，所以猜测flag在这个文件内

5. 访问ip:port/f1ag_1s_h3re.php,内容如下：

   ```html
   <html>
       <head></head>
       <body>
           cyberpeace{da5349b636d205f4dfbf15171bdd73b3}
       </body>
   </html>
   ```

6. 得出答案，cyberpeace{da5349b636d205f4dfbf15171bdd73b3}

7. 提交答案，正确

## 独立思考

### 1. robots协议有哪些规则？

下面是百度的robots.txt文件，

```txt
User-agent: Baiduspider
Disallow: /w?
Disallow: /client/
Disallow: /divideload/
Disallow: /edit/
Disallow: /l/
Disallow: /redirect/
Disallow: /reference/
Disallow: /search

User-agent: Googlebot
Disallow: /update
Disallow: /history
Disallow: /usercard
Disallow: /usercenter
Disallow: /client/
Disallow: /divideload/
Disallow: /edit/
Disallow: /l/
Disallow: /redirect/
Disallow: /reference/

User-agent: *
Disallow: /
```

可以看出它先列出来一些User-agent，并对各个User-agent分别设置了不允许爬取的路径，在文件的最后使用一个*来代表文件前面没提到的User-agent，并对这些User-agent统一限制了不允许爬取的路径。

除了这些字段，其实还有Sitemap字段(用于优化本网站被爬虫的爬取)、Allow字段(用于告诉哪些可以爬)

当然，robots协议只是约定俗成的，并不是一个规范，也不能用来保证网站的隐私，爬虫可以忽略robots.txt进行爬取网站。

## 产生过的疑问

1. robots协议有哪些规则？
