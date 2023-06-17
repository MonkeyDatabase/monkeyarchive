---
layout: post
title: Web新手题-xff_referer
excerpt: "攻防世界新手Web题目-xff_referer"
date:   2020-10-10 16:41:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“X老师告诉小宁其实xff和referer是可以伪造的。”
2. 访问网站，显示“ip地址必须为123.123.123.123”
3. 使用Burp Suite添加请求头字段，X-Forwarded-For:123.123.123.123
4. 网页显示，“必须来自https://www.google.com”
5. 使用Burp Suite添加请求头字段，Referer:https://www.google.com
6. 发现Flag，cyberpeace{4a9e44a98041b280067cff904229559e}
7. 提交，答案正确

## 独立思考

### 1. HTTP请求头有哪些常用且具有特殊意义的字段？

| 字段名          | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| X-Forwarded-For | 代表HTTP请求端的真实IP，只有在通过HTTP代理或者负载均衡时才会添加该项，因为代理会在应用层接收该请求，重新以自己的IP发送，会丢失真实客户端IP |
| Referer         | 包括协议、域名、端口、查询参数，代表从哪个网页跳转过来，服务器端可以统计流量、广告统计等等 |
| Origin          | 包括协议、域名、端口，一般用于CORS跨域请求                   |
| Host            | 包括域名、端口                                               |

## 产生过的疑问

1. HTTP请求头有哪些常用且具有特殊意义的字段？
