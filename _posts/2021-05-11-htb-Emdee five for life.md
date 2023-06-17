---
layout: post
title: HTB-Easy-Emdee five for life
excerpt: "第二道HTB题目,解题过程"
date:   2021-05-11 17:03:00
categories: [HackTheBox]
comments: true
---

## 解题步骤

1. 访问目标网页

   ![image-20210511170428315](https://monkeydatabase.github.io/img//image-20210511170428315.png)

2. 题目描述：Can you encrypt fast enough?

3. 检测Cookie，发现存在一个key为PHPSESSID的Cookie，将其删除之后刷新页面，虽然该Cookie重新生成了，但是值发生了改变，对其进行编解码均无明显特征

4. 对网页上给出的字符串进行MD5散列，提交之后显示Too slow，而且刷新了要进行MD5散列的字符串

   ![image-20210511170838316](https://monkeydatabase.github.io/img//image-20210511170838316.png)

5. 猜测可能通过请求体进行的时间判断，使用Burp Suite进行抓包，发现并没有异常

   ```http
   POST / HTTP/1.1
   Host: 138.68.183.83:30305
   Referer: http://138.68.183.83:30305/
   Content-Type: application/x-www-form-urlencoded
   Content-Length: 37
   Cookie: PHPSESSID=nsqgk152dlh578ojnd4hh0rfb4
   Upgrade-Insecure-Requests: 1
   
   hash=4a6ca4570c451b080cec591d374b55a1
   ```

6. 可能在服务器侧使用Cookie存储了上一次字符串生成时间与本次提交时间进行了时间差计算

7. 尝试使用Python脚本进行MD5

   * 代码

      ```python
      import requests
      from lxml import etree
      import hashlib
      
      if __name__ == '__main__':
          url = 'http://138.68.183.83:30305/'
          session = requests.session()
          resp = session.get(url)
          source_str = etree.HTML(resp.text).xpath("/html/body/h3")[0].text
          md5_str = hashlib.md5(source_str.encode(encoding='UTF-8')).hexdigest()
          data = {'hash': md5_str}
          print(data)
          resp = session.post(url, data=data)
          print(resp.cookies, resp.content)
      ```
   
      
   
   * 响应
   
     ```html
     <html>
      <head></head>
      <body style="background-color:powderblue;">
       <title>emdee five for life</title>
       <h1 align="center">MD5 encrypt this string</h1>
       <h3 align="center">4jqN3uwchnozTU1DH51U</h3>
       <p align="center">HTB{N1c3_ScrIpt1nG_B0i!}</p>
       <center>
        <form action="" method="post">
         <input type="text" name="hash" placeholder="MD5" align="\'center\'" />
         <br />
         <input type="submit" value="Submit" />
        </form>
       </center>
      </body>
     </html>
     ```
   
8. 因此，结果为HTB{N1c3_ScrIpt1nG_B0i!}

## 独立思考

本题较简单

## 产生过的疑问

本题较简单

