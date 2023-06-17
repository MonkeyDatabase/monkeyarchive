---
layout: post
title: 浏览器被限定，如何解决
excerpt: ""
date:   2020-09-04 12:00:00
categories: [WEB]
comments: true
---

## 学习笔记

### 1、请在微信客户端打开

#### 第一次尝试 尝试修改UA

UA是微信自己定义的

```
Mozilla/5.0 (Windows NT 6.1; WOW64) 
AppleWebKit/537.36 (KHTML, like Gecko) 
Chrome/53.0.2785.116 
Safari/537.36 QBCore/4.0.1301.400 
QQBrowser/9.0.2524.400 
Mozilla/5.0 (Windows NT 6.1; WOW64) 
AppleWebKit/537.36 (KHTML, like Gecko) 
Chrome/53.0.2875.116 
Safari/537.36 
NetType/WIFI 
MicroMessenger/7.0.5 WindowsWechat
```

部分情况，可以正常访问网页，获取源代码

然而还有部分会返回微信的处理逻辑

```html
<!DOCTYPE html>
<html>
    <head>
        <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=0">
    </head>
    <body>
        <script type="text/javascript">
            var ua = navigator.userAgent.toLowerCase();
            var isWeixin = ua.indexOf('micromessenger') != -1;
            var isAndroid = ua.indexOf('android') != -1;
            var isIos = (ua.indexOf('iphone') != -1) || (ua.indexOf('ipad') != -1);
            if (!isWeixin) {
                document.head.innerHTML = '<title>抱歉，出错了</title><meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=0"><link rel="stylesheet" type="text/css" href="https://res.wx.qq.com/open/libs/weui/0.4.1/weui.css">';
                document.body.innerHTML = '<div class="weui_msg"><div class="weui_icon_area"><i class="weui_icon_info weui_icon_msg"></i></div><div class="weui_text_area"><h4 class="weui_msg_title">请在微信客户端打开链接</h4></div></div>';
            }
        </script>
    </body>
</html>
```

#### 第二次尝试 Chrome driver模拟操作

1. 使用User-Agent Switcher for Chrome修改UA
2. 使用Edit this cookie添加Cookie
3. python使用selenium和ChromeDriver驱动Chrome

```python
from selenium import webdriver

#生成一个Chrome浏览器的driver，使用时相当于requests库
driver = webdriver.Chrome()

#发起请求
driver.get('http://xxx.xxx.xxx')

#打印源码
print(driver.page_source)

#通过id获取全局唯一标签
xxx = driver.find_element_by_id('xxxxx')

#通过在Chrome浏览器中对应标签上右键Copy的XPath进行检索获取
xxx = driver.find_element_by_xpath('xxxxx')

#点击按键
xxx.click()

#输入框填充内容
xxx.send_keys()
```

> 要尽量不让浏览器检测到受到自动测试软件的控制
>
> 因为直接实例化的driver对象是最原始的浏览器对象，需要加上一些测试的配置
>
> 我们手动启动Chrome，开放一个端口用于远程调试
>
> ```powershell
> cd C:\Program Files (x86)\Google\Chrome\Application\
> chrome.exe   --remote-debugging-port=9999
> ```
>
> 在python中配置连接目标端口的Chrome程序，这样启动的程序会包含原有的Cookie和所有扩展及设置
>
> ```python
> option=webdriver.ChromeOptions()
> option.binary_location = 'C:\Program Files (x86)\Google\Chrome\Application\chrome.exe'
> option.add_experimental_option('debuggerAddress','127.0.0.1:9999')
> 
> driver=webdriver.Chrome(executable_path='C:\Program Files (x86)\Google\Chrome\Application\chromedriver.exe',chrome_options=option)
> 
> ```
>
> 部分网页会通过js检查浏览器中是否有chromedriver，通过的是chromedriver中部分指纹，例如cdc，可以将其修改但不影响程序执行，即可绕过检测机制。

但是这个也不是长久之计，因为每次的Cookie都可能会过期，虽然只要在driver运行期间每次用掉一个Cookie会再获得一个新的，但是当是总会在某一个时间点服务器会吊销之前所有的Cookie强制让用户重新通过微信公众平台的单点登录重新申请，微信是采用Oauth2使用openid来获取Cookie的，这部分使用Burp代理微信(微信登陆的时候可以选择代理)来看到部分过程，但是关键步骤的数据包的请求体和响应体被加密了，是微信自己在通信前进行的加密，所以只能想其他办法或者找到这部分的交互原理，至少这里面包括了openid这一项内容。

## 独立思考

### 1. response中的中文乱码，如何解决？

```py
	response=session.get(searchurl,headers=headers,timeout=3)
    f=open('cookie','w',encoding='utf-8')
    f.write(response.text)
    f.close()
```

## 产生过的疑问

1. response中的中文乱码，如何解决？