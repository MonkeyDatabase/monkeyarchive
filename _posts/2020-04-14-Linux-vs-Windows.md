---
layout: post
title:  "Linux vs Windows"
date:   2020-04-14 16:00:00
excerpt: "Linux和Windows有什么区别，Linux一定更安全嘛"
categories: [CyberSecurity]
comments: true
---

操作系统可以是结构单一的、层次的、基于微核的，其中基于微核的是最安全的。

## Linux 和 Windows 操作系统主要的安全特性 

| 类别 | 特性 | Linux | Windows | 优势方 |
|:--------|:-------:|:--------:|:-------:|:--------:|
| 基本安全 | 验证、访问控制、加密、日志   | 可插入的认证模块、插件模块、Kerberos、PKI、Winbind、ACLs、LSM、SELinux、受限的访问保护实体检测、内核加密 | Kerberos、PKI、访问控制列表、受限的访问实体检测、微软的应用程序安全程序接口 | Linux |
| 网络安全与协议 | 验证层、网络层 | OpenSSL、OpenSSH、OpenLDAP、IPsec | SSL、SSH、LDAP、AD、IPsec | 平分秋色 |
| 应用安全 | 防病毒、防火墙、入侵安全检测、Web服务器、Email服务器、智能卡支持 | OpenAV、Panda、TrendMicro、内核内建的防火墙、Snort、Apache、sendemail、Postfix、PCKS 11、exec-shield | Mcafee、Symantec、CheckPoint、IIS、Exchange、Outlook、PCKS 11 | Linux |
| 分发和操作 | 安装、配置、管理、加固、漏洞扫描器 | 安装和配置工具、Bastille、大部分操作通过命令行完成、Nessus、发行版相关的Up2Data、Yast、Webmin | Windows自带的安装和配置工具、没有特定的加固工具、管理GUI、使用默认安装的配置 | 平分秋色 |
| 确信度 | 常见的公共标准证书、缺陷管理 | Linux达到了EAL3，有较好的缺陷处理能力 | Windows达到了EAL4，有较好的缺陷处理能力 | Windows |
| 可信计算 | 可信平台的模块、可信计算软件栈、工具、验证 |   |    |    |
| 开放标准 | IPsec、POSIX、常见标准 | Linux遵循所有的开源协议 | 微软参与了一些开源标准，但仍有一部分私有标准 | Linux |