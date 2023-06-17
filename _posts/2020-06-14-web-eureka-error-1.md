---
layout: post
title: eureka exit with 1
excerpt: "eureka启动时直接退出，exit with 1"
date:   2020-06-14 16:53:00
categories: [WEB]
comments: true
---

## Eureka简介

Eureka用于微服务注册中心。各个微服务之间基于http互相调用，而http通过ip地址和端口号唯一标识一个进程，所以每个微服务需要其他微服务进程所在的ip及端口号才能进行调用。当ip地址和端口号发生改变时，每个微服务都应知道其新的ip地址和端口号，为了降低耦合，所有的微服务向Eureka注册自己的ip地址、端口等信息，便于各个服务之间的相互调用。

## Exit with 1

### 原因一

可能是Eureka的版本与其它依赖不兼容，尝试更换Eureka版本

### 原因二

可能是配置文件出现问题，.yml文件中每个条目的格式为[key: value]，冒号后面要有一个空格

### 原因三

可能是端口冲突，更换端口

### 原因四

pom.xml中依赖的包名是否有误，检查有无输入错误

### 原因五

重启233~
