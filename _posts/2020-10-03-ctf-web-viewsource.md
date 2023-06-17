---
layout: post
title: Web新手题-view_source
excerpt: "攻防世界新手Web题目-view_source"
date:   2020-10-03 21:23:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 查看题目描述“X老师让小宁同学查看一个网页的源代码，但小宁同学发现鼠标右键好像不管用了”

2. 打开网页发现网页仅有一个字符串：FLAG is not here

   * 发现字符串无法选中
   * 鼠标右键无法起作用

3. 点击F12调用开发者工具，查看网页源代码

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <title>Where is the FLAG</title>
   </head>
   <body>
   <script>
   document.oncontextmenu=new Function("return false")
   document.onselectstart=new Function("return false")
   </script>
   
   
   <h1>FLAG is not here</h1>
   
   
   <!-- cyberpeace{d4923a1f9bf796bee848ecef818ced07} -->
   
   </body>
   </html>
   ```

4. 发现flag：cyberpeace{d4923a1f9bf796bee848ecef818ced07}

5. 提交，答案正确

## 独立思考

### 1. 为什么它可以禁用选择和右键？

通过浏览代码可以发现：

```html
<script>
	document.oncontextmenu=new Function("return false")
	document.onselectstart=new Function("return false")
</script>
```

这是通过DOM对象的事件对象进行的屏蔽，因为对这个事件绑定的事件对象直接return false了。

1. 文档节点：HTML文档
2. 元素节点：HTML标签
3. 属性节点：HTML元素的属性
4. 文本节点：插入HTML的标签中间的文本块
5. 注释节点：一个注释块

HTML DOM事件允许JavaScript在HTML文档的**元素节点**上注册事件，事件通常与函数结合使用，函数不会在事件发生前执行。

> 遇到不熟悉的事件可以在[事件对象手册](https://www.runoob.com/jsref/dom-obj-event.html)中查找

### 2. HTML、CSS、JavaScript在同一个document中的加载解析顺序？

* 总体来看，按照HTML文档的文件流按顺序进行解析，先解析head再解析body
* 遇到引用的外部文件，会进行下载，且下载进程并不会被打断
* 遇到script标签，会打断渲染进程将控制权交给JavaScript解析器，解析执行完成后，再将控制权交还给渲染器进程

通过上面的描述，script代码写在body的最后比较好，因为很可能JavaScript想要处理DOM节点时，该DOM节点还没有被渲染进程解析，很有可能报”undefined“错误。

不过这样，body标签内部将十分混乱，后来人们提出了另外的解决方法：

| 解决方案 | window\.onload                                               | \$(document).ready()                      |
| -------- | ------------------------------------------------------------ | ----------------------------------------- |
| 执行时机 | 必须等待网页全部加载完毕，包括图片等大文件，然后再执行window.onload再执行代码 | 只需要等待网页的DOM结构加载完毕，就能执行 |
| 执行次数 | 只能执行一次，如果执行第二次，第一次的执行结果会被覆盖       | 可以执行多次，各次之间无关联              |
| 简写方案 | 无                                                           | \$(function(){<br/><br/>})                |
| 方法来源 | JavaScript                                                   | jQuery                                    |

### 3. 如何让已执行的JavaScript代码失效？

因为JavaScript代码在被解析时会被执行，到最后所有的代码都被执行，网页到达complete状态，此时类似于该题中的禁用右键和选中的事件对象已经被注册完成，此时再去删除HTML代码中的script已经没有了意义，因为它们已经是被注册过的了。

如果要让它们失效，也就是要让它们的事件函数与元素节点解除绑定，需要通过F12的开发者工具中的Console操作document对象。

例如本题中的禁用右键，只需在控制台中执行*document.oncontextmenu=null*,就会在内存中解除掉注册，从而可以继续使用右键功能。

### 4. html注释有哪几种？

无论是单行注释还是多行注释，HTML注释的标签只有一种，那就是\<\!\-\-           \-\-\>

## 产生过的疑问

1. 为什么它可以禁用选择和右键？
2. HTML、CSS、JavaScript在同一个document中的加载解析顺序？
3. 如何让已执行的JavaScript代码失效？
4. html注释有哪几种？