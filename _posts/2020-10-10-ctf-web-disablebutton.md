---
layout: post
title: Web新手题-disabled_button
excerpt: "攻防世界新手Web题目-disabled_button"
date:   2020-10-10 12:19:00
categories: [CTF]
comments: true
---

## 我的解题过程

1. 题目描述为“X老师今天上课讲了前端知识，然后给了大家一个不能按的按钮，小宁惊奇地发现这个按钮按不下去，到底怎么才能按下去呢？”
2. 按钮不能点，是因为\<input\>标签有disable属性，在F12开发者工具->Elements中把这个属性删掉即可
3. 点击按钮，界面出现cyberpeace{303ee9547a65c6e333f029dfc88231e3}
4. 提交答案，正确

## 独立思考

### 1. 还有什么其他方法找到不能点击的按钮的点击效果？

该input标签的所有特征如下：

```html
<form action="" method="post">
	<input disabled="" class="btn btn-default" style="height:50px;width:200px;" type="submit" value="flag" name="auth">
</form>
```

form表单允许用户在表单中输入内容，比如：文本、下拉菜单、单选框、复选框等等。

* 表单标签通常为\<input\>
* 表单标签的类型用type属性来定义，通常有text、password、select、radio、checkbox等
* 表单标签的键值对为name：value

所以通过阅读该form表单的情况，可以使用Burp Suite的Repeater发起请求：

1. 使用get访问网页，Burp Suite的Proxy截包
2. 发送到Repeater改包，请求方式改为POST，请求体改为*auth=flag*，点击发送
3. 在Repeater的Response中就可以看到flag

### 2. 如果flag在按钮的js代码中，如何快速定位？

F12打开开发者工具->Elements面板，这个面板分为左右两部分，左侧用于看源码，右侧有一个EventListener，在网页中想要分析的按钮上邮件点击“检查”，EventListener就可以显示出该事件的EventListener，进而跳转到对应脚本的对应行。

## 产生过的疑问

1. 还有什么其他方法找到不能点击的按钮的点击效果？
2. 如果flag在按钮的js代码中，如何快速定位？

