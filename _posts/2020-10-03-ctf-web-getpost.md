---
layout: post
title: Web新手题-get_post
excerpt: "攻防世界新手Web题目-get_post"
date:   2020-10-03 22:49:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“X老师告诉小宁同学HTTP通常使用两种请求方法，你知道是哪两种吗？”

2. 使用Burp Suite捕获网络请求，查看效果

3. 浏览器输入目标网址，点击访问，Burp Suite捕获请求，直接发送到Repeater

4. Burp Suite的Response对中文解析乱码，在User options->Display->Http Message Display,把字体改成中文字体即可。

5. 网页正文为“请用GET方式提交一个名为a,值为1的变量”

   ```http
   GET / HTTP/1.1
   Host: 220.249.52.133:38339
   User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:73.0) Gecko/20100101 Firefox/73.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
   Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
   Connection: close
   Upgrade-Insecure-Requests: 1
   Cache-Control: max-age=0
   ```

6. 将Repeater的Request中的请求报文的请求行，加上/?a=1

   ```http
   GET /?a=1 HTTP/1.1
   Host: 220.249.52.133:38339
   User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:73.0) Gecko/20100101 Firefox/73.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
   Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
   Connection: close
   Upgrade-Insecure-Requests: 1
   ```

7. 这次网页多了一行"请再以POST方式随便提交一个名为b,值为2的变量"

8. 使用Burp Suite的Repeater修改请求方式为POST，并在请求体中加上一个值为2的b字段

   ```http
   POST /?a=1 HTTP/1.1
   Host: 220.249.52.133:38339
   User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:73.0) Gecko/20100101 Firefox/73.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
   Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
   Connection: close
   Upgrade-Insecure-Requests: 1
   Cache-Control: max-age=0
   Content-Length: 3
   Content-Type: application/x-www-form-urlencoded
   
   b=2
   ```

9. 发现flag：cyberpeace{e3a724bda5b72bbddf76c3ca8a484b7e}

10. 提交，答案正确

## 独立思考

### 1. Burp Suite为什么会中文乱码？

Burp Suite展示数据包时中文乱码其实并不是解码失败的问题，而是它默认选用了英文字体，它的字库里没有中文，所以只需在User options->Display->Http Message Display,把字体改成中文字体即可。

### 2. 通过Burp Suite手动添加GET参数有什么需要注意的点？

1. 请求行格式：请求方法 URL字段 HTTP协议版本 CRLF

2. 当访问网站主页时，即浏览器只需输入网站域名或者ip:port时，请求行为请求方式、/、HTTP协议版本号(HTTP/1.1)，此处的"/"代表了访问网站根目录，即主页

   ```http
   GET / HTTP/1.1
   ```

3. 当访问某一URL时，若要带上参数，只需在请求行的URL路径后跟上*?x=xxx&y=yyy*，?用来标识GET参数的开始，之后每个键值对用\&作为分割。

4. 注意访问网站根目录时，原请求行URL字段是有一个"/"的，不要把它误当作HTTP协议版本字段。所以正确格式如下：

   ```http
   GET /?x=xxx&y=yyy HTTP/1.1
   ```

### 3. 通过Burp Suite手动添加POST参数有什么需要注意的点？

* 首先需要将请求行的请求方式改为POST
* 在请求头部分，添加*Content-Type: application/x-www-form-urlencoded*，表明MIME类型为form表单
* 在请求体部分，写入要传输的字段如*x=xxx&y=yyy*，用&作为每个键值对的分隔符

## 产生过的疑问

1. Burp Suite为什么会中文乱码？
2. 通过Burp Suite添加GET参数有什么需要注意的点？
3. 通过Burp Suite添加POST参数有什么需要注意的点？

