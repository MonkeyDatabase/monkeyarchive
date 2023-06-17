---
layout: post
title: Web新手题-weak_auth
excerpt: "攻防世界新手Web题目-weak_auth"
date:   2020-10-10 13:12:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“小宁写了一个登陆验证页面，随手就设了一个密码。”
2. 登入页面，有一个两个输入框用于输入账号密码，有两个按钮login和reset，reset用于清空两个输入框
3. 输入admin、123456，点击登录成功
4. 发现flag，cyberpeace{034f5e235569ff4efd3066a110ce0b9b}

## 独立思考

### 1. 如果是爆破弱口令，如何使用Burp Suite这个工具？

1. 使用Proxy开启截包
2. 在网站上随意输入账号密码，点击登录
3. Proxy截包后，转发给Intruder
4. 在Intruder中添加用户名标记和密码标记
5. 选用集束炸弹模式(Cluster bomb)，第一个payload选Username的payload，第二个payload选择Password的payload
6. 执行后，对Response的Length进行排序，一般密码正确时返回的Length不同

若只爆破弱口令，则可以选择Sniper模式。

## 产生过的疑问

1. 如果是爆破弱口令，如何使用Burp Suite这个工具？

