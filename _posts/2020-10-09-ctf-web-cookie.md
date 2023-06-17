---
layout: post
title: Web新手题-Cookie
excerpt: "攻防世界新手Web题目-Cookie"
date:   2020-10-09 16:12:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述为“X老师告诉小宁他在cookie里放了些东西，小宁疑惑地想：‘这是夹心饼干的意思吗？’”

2. 使用EditThisCookie扩展，查看该网站使用的Cookie，发现有一个值为cookie.php、名为look-here的Cookie。

   ```cookie
   [
   {
       "domain": "xxx.xxx.xxx.xxx",
       "hostOnly": true,
       "httpOnly": false,
       "name": "look-here",
       "path": "/",
       "sameSite": "unspecified",
       "secure": false,
       "session": true,
       "storeId": "0",
       "value": "cookie.php",
       "id": 1
   }
   ]
   ```

3. 访问ip:port/cookie.php，网页正文为“See the http response”

4. 通过F12打开开发者工具，通过Ctrl+R刷新网页，发现cookie.php的响应头有问题

   ```http
   HTTP/1.1 200 OK
   Date: Fri, 09 Oct 2020 08:14:50 GMT
   Server: Apache/2.4.7 (Ubuntu)
   X-Powered-By: PHP/5.5.9-1ubuntu4.26
   flag: cyberpeace{ff50741d8f89a6780724bdeb125ce36b}
   Vary: Accept-Encoding
   Content-Encoding: gzip
   Content-Length: 253
   Keep-Alive: timeout=5, max=100
   Connection: Keep-Alive
   Content-Type: text/html
   ```

5. 发现flag，cyberpeace{ff50741d8f89a6780724bdeb125ce36b}

6. 提交答案，正确

## 独立思考

### 1. 不使用EditThisCookie扩展的话，如何查看Cookie？ 

1. 在F12开发者工具Console中*document.cookie*，可回显Cookie，但是只能回显没有设置HTTP Only的Cookie
2. 在Chrome浏览器地址栏左侧有一个按键，可以查看Cookie

### 2. Cookie的各个字段有什么含义？

* Name/Value：设置Cookie的名称及对应的值
* Expires：设置Cookie的生存期
  * 会话型Cookie，Expire属性缺省，仅被保存在客户端内存中，在用户关闭浏览器时失效
  * 持久型Cookie，生存期结束或触发某些销毁Cookie的事件才会失效，被保存在客户端硬盘中
* Path：设置可以访问该Cookie的Web站点
* Domain：指定可以访问该Cookie的Web站点或域，Cookie并未遵守同源策略(例如网页知乎会使用百度的Cookie)
* Secure flag：指定是否使用HTTPS安全协议发送Cookie。在HTTPS建立阶段，浏览器会检查Web网站的SSL证书有效性，如果无效，浏览器并不会立即终止用户连接请求，而是显示安全风险
* HTTP only flag：用于防止客户端脚本通过document.cookie属性访问Cookie，有助于保护Cookie不被跨站脚本窃取或篡改，js无法修改这个值。
* HOST only flag：当Cookie无Domain字段、Domain字段为空、Domain字段不合法时为true

### 3. Cookie与Session有什么关系？

Cookie是保存在客户机中的简单的文本文件，由一个名称、一个值、其他几个用于控制Cookie有效期、安全性、适用范围的可选属性组成。

Session是为了让服务器知道请求者是谁，为此服务给给每个客户端分配不同的身份标识，然后每次请求，客户端向服务器发起请求时带上这个身份标识，服务器就可以知道是谁了。

Session在客户端通常采用Cookie作为存储机制。

### 4. Session有什么缺陷？

* 可扩展性差：如果Web服务器作了负载均衡，如果请求到另一台机器时session会消失，若保持所有机器维护同一个session记录会极大增加开销。
* 内存开销：当访问量越大时，存储Session的开销也就越大。
* CSRF：很容易受到跨站请求伪造攻击。

### 5. Token有什么用？

用户登陆系统，系统给用户发放一个令牌(token)。token里面包含了用户的userID，为了防止伪造，还在其中服务器用密钥和加密算法对数据的签名，签名和数据一同组成token。

这样就可以解决Session导致的扩展性差的问题，因为服务器端无需存储token，只需将用户发来的数据使用同样的方法进行加密与签名进行比较，如果相同，就直接取userID。

## 产生过的疑问

1. 不使用EditThisCookie扩展的话，如何查看Cookie？
2. Cookie的各个字段有什么含义？
3. Cookie与Session有什么关系？
4. Session有什么缺陷？
5. Token有什么用？

