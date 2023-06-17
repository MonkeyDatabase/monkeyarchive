---
layout: post
title: Chrome扩展
excerpt: "Chrome扩展其实就是一个WebApp，包括一些html、js、css及其他一些可选的静态资源"
date:   2020-06-29 8:51:00
categories: [WEB]
comments: true
---

## 架构

该架构纯粹是为了清晰，自行规定的。manifest.json中用相对路径定义了各个文件的位置，因此除了manifest.json要放在最上层，其它的文件可以任意存放。

* manifest.json：用于描述整个扩展架构的配置文件
* html文件夹：用于存放html页面
  * 不可视界面
    * background.html中包含了Chrome扩展的主要业务逻辑，包含两种分别是persistent background pages和event pages(这两种在manifest中配置区别在于persistent参数为true还是false)
  * 可视界面(browser actions与page actions互斥)
    * browser actions，适用于所有界面，图标位于浏览器地址栏外的右侧
    * page actions，只能作用于特定界面，图标位于浏览器地址栏内的右侧
    * context menu，右键菜单
    * options，支持用户定制Chrome扩展的参数
    * override，可以替换掉浏览器中打开的默认界面，如[chorme://bookmarks](chorme://bookmarks)，[chrome://history](chrome://history)，[chrome://newtab](chrome://newtab)。一个chrome扩展限制仅能替换一个页面。
  * chrome.extension.getViews()、chrome.extension.getBackgroundPage()等方法可以获取扩展页面的window对象，可以对其它页面的DOM进行任意操作。
* js文件夹：用于存放js代码
  * content scripts，可以实现扩展与用户打开的Web界面的交互，是一组js文件，运行在浏览器当前打开页面的上下文中，可以读取并修改当前打开页面的DOM
  * XMLHttpRequest，实现扩展与其他服务器的交互
  * 除了js的通用API之外，还可调用chrome.* API
* css文件夹：用于存放css样式
* img：用于存放程序图标等图片素材

## manifest.json

* manifest.json是一个Web应用程序清单，用json文件提供应用程序的信息。
* manifest.json只在加载时读取一次，对manifest文件的修改必须重新加载才能生效。
* manifest_version属性必须为2
* browser_action用于启用browser actions，icon必须，tooltip、badge、popup可选，popup是一个点击图标时出现的弹出框。
* page_action类似于browser_action，但没有badge，有显示和隐藏的变化，只在指定的tab显示图标且在地址栏内。
* background可设置scripts，自动生成后台html，也可以手写html。
* chorme_url_overrides的属性可以为bookmarks、history、newtab，不过每个扩展只能重写一个页面。(一般不用这个功能)

```json
{
  "manifest_version": 2,
  "name": "Diablo",
  "version": "0.1.1",
  "description": "A Group of my tools",
  "browser_action": {
    "default_icon": "img/myicon.png",
    "default_title": "Diablo",
    "default_popup": "html/popup.html"
  },
  "options_ui": {
    "page": "html/options.html",
    "chrome_style": true,
    "open_in_tab": true
  },
  "background": {
    "scripts": ["js/background.js"]
  },
  "chrome_url_overrides": {
    "newtab": "html/popup.html"
  },
  "permissions":["downloads",
    "alarms",
    "contextMenus",
    "cookies",
    "notifications",
    "storage",
    "tabs",
    "webRequest",
    "webRequestBlocking",
    "http://*/*", "https://*/*"
  ]
}
```

## 存储机制

Chrome可以使用以下三种任意一种:

* H5的localStorage API实现本地存储
* Google的chrome.storage.* API实现浏览器存储
* Google的chrome.cookies.* API实现的cookie存储

### 1、chrome.storage.*

申请权限

```json
"permissions": [
          "storage"
        ],
```

使用方式

* chrome.storage.sync,开启数据同步，数据同步到google帐号
* chrome.storage.local,用户禁用数据同步时，仅本地使用
* chrome.storage.managed，域管理员可以存取，扩展仅能读

码

```js
//存储一或多个数据
//此例中item为key-value对，items是一组item
chrome.storage.sync.set(object items,function(){})
//通过key查询value
//传入的参数可以是单个key，也可以是多个key
chrome.storage.sync.get(string or array of string or object keys, function(object items) {...})
//通过key删除key-value
chrome.storage.sync.remove(string or array of string keys, function() {...})
//清空所有key-value
chrome.storage.sync.clear(function(){...})
//获取已用存储空间，B为单位，第一个参数为null返回全部
chrome.storage.sync.getByteInUse(string or array of string keys, function(integer bytesInUse) {...})
//对某些敏感数据监听
chrome.storage.sync.onChanged.addListener(function(changes,namespace){})
```

### 2、chrome.cookies.*

申请权限

```json
"permissions": [
	"cookies",
	"*://*.google.com"
]
```

码

```javascript
//条件查询cookie
//details必选字段有url、name，可选字段有storeId
chrome.cookie.get(object details,function(Cookie cookie){...})
//获取CookieStore中所有的Cookie对象
chrome.chrome.cookies.getAll(object details, function(array of Cookie cookies) {...})
//设置Cookie
chrome.cookies.set(object details, function(Cookie cookie) {...})
//删除Cookie
chrome.cookies.remove(object details, function(object details) {...})
//获取所有Cookie仓库
chrome.cookies.getAllCookieStores(function(array of CookieStore cookieStores) {...})
//监听Cookie对象变化
chrome.cookies.onChanged.addListener(function(object changeInfo) {...})
```



