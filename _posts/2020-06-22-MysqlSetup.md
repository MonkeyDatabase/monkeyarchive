---
layout: post
title: Mysql安装
excerpt: "Mysql在Docker容器中的安装"
date:   2020-06-22 8:20:00
categories: [WEB]
comments: true
---



拉取镜像

```shell
docker pull mysql:5.7
```

利用镜像创建容器

```shell
docker run -d -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
```

如果此时web工程仍访问不到项目，开启外部访问

```shell	
docker exec -it thismysqlname /bin/bash
mysql -u root -p
use mysql
# %代表所有主机 root代表远程访问帐户 123456为远程访问所用的密码
ALTER USER'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
# 刷新配置信息
flush privileges;
```

开放3306端口

```shell
firewall-cmd --zone=public --add-port=3306/tcp --permanent
```

设置为随docker启动而自启

```shell
docker update restart=always mysql
```

