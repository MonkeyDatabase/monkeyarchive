---
layout: post
title: Web进阶题-easytornado
excerpt: "攻防世界进阶Web题目-easytornado"
date:   2020-10-24 19:07:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述，"Tornado框架"

2. 访问目标网站，网页内容如下

   ```html
   <body>
       <a href="/file?filename=/flag.txt;filehash=cc8cddc19c52840b5cf02fae58d8adc2">/flag.txt</a>
       <br/>
       <a href="/file?filename=/welcome.txt&filehash=e02a7dad157af2c4bb8839db8bca3524">/welcome.txt</a>
       <br/>
       <a href="/file?filename=/hints.txt&filehash=0ad4fefa43655b5470862f02611b7221">/hints.txt</a></body>
   ```

   * flag.txt

     ```txt
     /flag.txt
     flag in /fllllllllllllag
     ```

   * welcome.txt

     ```txt
     /welcome.txt
     render
     ```

   * hints.txt

     ```txt
     /hints.txt
     md5(cookie_secret+md5(filename))
     ```

3. 按照这些链接的href属性可以看出访问文件的大致格式为"/file?filename=文件名&filehash=文件散列值"，由flag.txt可以得出文件名为"/fllllllllllllag"，hint.txt应该是对文件散列值的提示，可是cookie_secret不知道从哪里找，因为这个网站并没有存储cookie到本地

4. 我们有这些文件的散列，但是由于一次md5就是128位再加上cookie_secret的长度，不可能从这些md5散列值倒推出cookie_secret

   * md5(cookie_secret+md5(/flag.txt))=cc8cddc19c52840b5cf02fae58d8adc2
   * md5(cookie_secret+md5(/hint.txt))=0ad4fefa4365to5b5470862f02611b7221
   * md5(cookie_secret+md5(/welcome.txt))=e02a7dad157af2c4bb8839db8bca3524

5. 当访问不存在的路径时，只会返回404:Not Found，所以404页面也没办法注入

6. 当访问文件的filehash不正确时，进入了"/error"页面

   * 有一个msg参数会输出到页面上
   * msg参数用户可控

7. 尝试注入error页面的msg参数

   * 输入"/error?msg=4*4"，页面返回ORZ

8. 由于之前做过的Flask渲染模板注入题目，加之"welcome.txt"中的render，开始研究渲染中的注入

   * 尝试"/error?msg=\{\{"".\_\_class\_\_\}\}"，网页返回ORZ，尝试多次后，发现**魔术方法**均不能使用，所以之前Flask的SSTI攻击方法就**不可用**了，无法通过子类、父类、实例化来进行调用os等模块，所以只能调用当前处理函数可以访问到的已经实例化的对象

   * 当访问到"/error?msg=\{\{handler\}\}",网页返回内容为"\_\_main\_\_.ErrorHandler object at 0x7fc20053e710>"

   * 在Pycharm中查看可利用的ErrorHandler的函数，发现它继承自RequestHandler

   * 查看ReuqestHandler源码，发现与settings方法

   * 访问"/error?msg=\{\{handler.settings\}\}"，网页没有返回ORZ，网页返回内容如下

     ```txt
     {'autoreload': True, 'compiled_template_cache': False, 'cookie_secret': '3b454adf-5caa-4fb8-8d78-b1df4c7af327'}
     ```

   * 成功获取到cookie_secret，3b454adf-5caa-4fb8-8d78-b1df4c7af327

9. 计算最终filehash

   * md5(/fllllllllllllag)=3bf9f6cf685a6dd8defadabfb41a03a1
   * cookie_secret+md5(/fllllllllllllag)=3b454adf-5caa-4fb8-8d78-b1df4c7af3273bf9f6cf685a6dd8defadabfb41a03a1
   * md5(cookie_secret+md5(filename))=5915a62b92ba31f67d058fe531b69846

10. 构造最终URL，"/file?filename=/fllllllllllllag&filehash=5915a62b92ba31f67d058fe531b69846"，网页返回内容如下

    ```txt
    /fllllllllllllag
    flag{3f39aea39db345769397ae895edb9c70}
    ```

11. 发现flag，flag{3f39aea39db345769397ae895edb9c70}

12. 提交，答案正确

## 独立思考

### 1. Tornado框架是什么？

Tornado是一个基于Python的Web服务框架和异步网络库，通过非阻塞I/O，可以承载成千上万的活动连接，完美的实现了长连接、WebSocket。

```python
#异步网络库
import tornado.ioloop
#Web框架，包括用来创建Web应用程序的RequestHandler等类
import tornado.web


class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello World")


def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
    ])


if __name__ == '__main__':
    app = make_app()
    app.listen(8888)
    tornado.ioloop.IOLoop.current().start()

```

### 2. Tornado中的cookie_secret是干什么用的？

Tornado的set_secure_cookie()和get_secure_cookie()方法用于设置和获取浏览器Cookie。

* set_secure_cookie()设置Cookie时会用cookie_secret对值进行HMAC签名，同时把时间戳添加到值中
* get_secure_cookie()从浏览器获取Cookie后，会使用cookie_secret进行签名校验，若签名不符或时间戳太旧，则认为cookie已被篡改

### 3. Tornado中有哪些可以利用的对象？

Tornado中的处理URL的对象为handler，这些handler类都继承自RequestHansdler。

* 在每次请求中，Application会根据注册将URL映射到Handler，并且每次请求会单独实例化一个对应的Handler对象。

  ```python
  def make_app():
      #传进去一个Handler与URL一一对应的Rule的数组
      return tornado.web.Application([
          (r"/",MainHandler),
      ])
  ```

* 对象的initialize被调用，传入的参数来自Application组态

  * RequestHandler的init函数

    ```python
    def __init__(
        self,
        application: "Application",#指向Application
        request: httputil.HTTPServerRequest,
        **kwargs: Any
    ) -> None:
        super(RequestHandler, self).__init__()
    
        self.application = application
        self.request = request
        self._headers_written = False
        ..........
    ```

  * RequestHandler的settings函数

    ```python
    @property
    def settings(self) -> Dict[str, Any]:
        """An alias for `self.application.settings <Application.settings>`."""
        return self.application.settings
    ```

* 对象的prepare()被调用

* 对象中一个HTTP方法被调用

* 处理完成后，on_finish()被调用

#### 最有效的找法

[官方文档](https://tornado-zh.readthedocs.io/zh/latest/guide/templates.html)的模板和UI模块给出了一些可以访问的对象

- escape: [tornado.escape.xhtml_escape](https://tornado-zh.readthedocs.io/zh/latest/escape.html#tornado.escape.xhtml_escape) 的别名
- xhtml_escape: [tornado.escape.xhtml_escape](https://tornado-zh.readthedocs.io/zh/latest/escape.html#tornado.escape.xhtml_escape) 的别名
- url_escape: [tornado.escape.url_escape](https://tornado-zh.readthedocs.io/zh/latest/escape.html#tornado.escape.url_escape) 的别名
- json_encode: [tornado.escape.json_encode](https://tornado-zh.readthedocs.io/zh/latest/escape.html#tornado.escape.json_encode) 的别名
- squeeze: [tornado.escape.squeeze](https://tornado-zh.readthedocs.io/zh/latest/escape.html#tornado.escape.squeeze) 的别名
- linkify: [tornado.escape.linkify](https://tornado-zh.readthedocs.io/zh/latest/escape.html#tornado.escape.linkify) 的别名
- datetime: Python [datetime](https://docs.python.org/3.4/library/datetime.html#module-datetime) 模块
- handler: 当前的 [RequestHandler](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.RequestHandler) 对象
- request: [handler.request](https://tornado-zh.readthedocs.io/zh/latest/httputil.html#tornado.httputil.HTTPServerRequest) 的别名
- current_user: [handler.current_user](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.RequestHandler.current_user) 的别名
- locale: [handler.locale](https://tornado-zh.readthedocs.io/zh/latest/locale.html#tornado.locale.Locale) 的别名
- _: [handler.locale.translate](https://tornado-zh.readthedocs.io/zh/latest/locale.html#tornado.locale.Locale.translate) 的别名
- static_url: [handler.static_url](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.RequestHandler.static_url) 的别名
- xsrf_form_html: [handler.xsrf_form_html](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.RequestHandler.xsrf_form_html) 的别名
- reverse_url: [Application.reverse_url](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.Application.reverse_url) 的别名
- 所有从 ui_methods 和 ui_modules Application 设置的条目
- 任何传递给 [render](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.RequestHandler.render) 或 [render_string](https://tornado-zh.readthedocs.io/zh/latest/web.html#tornado.web.RequestHandler.render_string) 的关键字参数

## 产生过的疑问

1. Tornado框架是什么？
2. Tornado中的cookie_secret是干什么用的？
3. Tornado中有哪些可以利用的对象？

