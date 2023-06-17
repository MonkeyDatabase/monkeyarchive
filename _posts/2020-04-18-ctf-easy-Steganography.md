---
layout: post
title: 简单的隐写
excerpt: "高校战疫网络安全分享赛-MISC-简单的隐写解题过程"
date:   2020-04-18 18:00:00
categories: [CTF]
comments: true
---

1. 文件损坏，可能是文件头不对，用HEX editor修改文件头0xffd8
2. 文件打开后，找不到信息。
   1. 用Stegsolve.jar分析图片通道
   2. 用HEX editor修改图片宽高
   3. 未发现宽高和文件颜色通道有问题
3. 用HEX editor检查jpeg文件尾，0xffd9发现其后仍有内容，将其copy至新文件
4. 发现新文件前2个字节为0x504b，为rar文件标识，将其另存为rar文件
5. 解压后发现ctf.txt，其内容为./.--./../-.././--/../-.-./.../../-/..-/.-/-/../---/-./---/..-./..-/-./../...-/./.-./.../../-/-.--/.--/.-/.-.
6. 摩斯电码解密后为EPIDEMICSITUATIONOFUNIVERSITYWAR
7. 用其成功解压flag.zip
8. flag.zip的flag.txt文件内容为VGgxc19pc19GbGFHX3lvdV9hUkVfcmlnSFQ=
9. 进行base64解密为Th1s_is_FlaG_you_aRE_rigHT
10. flag：flag{Th1s_is_FlaG_you_aRE_rigHT}