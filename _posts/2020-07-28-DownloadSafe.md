---
layout: post
title: 下载文件完整性校验
excerpt: "如何确定自己下载的文件是官方出版的，而没有被夹带私货？"
date:   2020-07-28 09:26:00
categories: [CyberSecurity]
comments: true
---

## 背景

通常情况下，我们需要到官方网站下载各种安装包，由于某些因素，某些网站下载速度不足，需要到下载站去下载文件，但是下载站下载的东西安全吗？会不会有别人脱壳之后加上了病毒程序？

此时，可以看到官方网站的下载按钮旁边一般会有一个MD5或者SHA，这个码可以用来确保文件的一致性，当我们下载下来的文件计算出来的哈希与官网上的哈希一致时，文件为原版文件。

## 操作流程

1. Windows+R输入CMD，启动小黑框
2. 输入certutil -hashfile
3. 把下载的文件拖入小黑框
4. 根据官网提供的码的类别，输入对应的MD5/SHA1/SHA256

> 最终样例如下，输入完成后，点击回车，就会计算出该文件的哈希码
>
> certutil -hashfile mysql-workbench-community-8.0.21-winx64.msi MD5
>
> 运行结果如下：
>
> MD5 的 mysql-workbench-community-8.0.21-winx64.msi 哈希:
> **bea0696dd7b8cbab25357ee8bf725639**
> CertUtil: -hashfile 命令成功完成。

与官网比对，发现为原版文件。

