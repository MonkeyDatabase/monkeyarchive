---
layout: post
title: 武汉加油
excerpt: "高校战疫网络安全分享赛-MISC-武汉加油解题过程"
date:   2020-04-18 10:00:00
categories: [CTF]
comments: true
---

这道题当时没做出来，一直以为需要脱VMP壳，做了一天也没做出来。

以下为看完官方的writeup的解题过程：

1. 图片大小4.51MB，里面一定有隐藏的文件。
2. JPG文件开始字节为FFD8，文件结束字节为FFD9，用Hex editor检索之后发现有多个FFD9。且第一个FFD9后面的开始字节为 20 61 72 21。
> [128个常见的文件头信息对照表](http://www.manongjc.com/article/56456.html)
3. 官方writeup采用了binwalk工具，它是用于搜索给定二进制文件以获取嵌入文件和代码的工具，用它的话，应该会比Hex editor手动检索更加准确便捷。
4. binwalk -e 1.jpg，提取出flag.exe文件和一个1FCB4.rar文件
5. flag.exe每输入6行，返回一个字符串。所以是输入6个特定的字符串会返回flag
> 当时我是用Hex editor提取出flag.exe，发现找不到那6个字符串，把这道题当作了逆向题，用od等工具测试，但是vmp壳脱不下来。
6. 官方writeup采用了Steghide，Steghide是一个可以将文件隐藏到图片或音频中的工具。

   {% raw %}
         steghide info <filename>
         steghide extract -sf <filename> -p <password>
   {% endraw %}
7. 提取出来一个flag.txt，发现六个字符分别为 ' 武 汉 加 油 ！
8. 输入flag.exe，得出flag : {zhong_guo_jia_you}
