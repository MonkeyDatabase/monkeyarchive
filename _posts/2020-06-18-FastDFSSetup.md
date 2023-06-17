---
layout: post
title: FastDFS安装
excerpt: "FastDFS在Docker中的搭建过程"
date:   2020-06-18 8:12:00
categories: [WEB]
comments: true
---



# 简介

FastDFS是一个开源的轻量级分布式文件系统，它对文件进行管理，功能包括：文件存储、文件同步、文件访问(文件上传、文件下载)等，解决了大容量存储和负载均衡的问题。

# 工作流程

## Dubbo

Dubbo属于SOA(Service-Oriented Architecture)架构，服务治理。Spring Cloud Alibaba集成了Dubbo，大大提高了微服务之间的数据访问效率。

通过IP找到服务器，通过端口找到程序，通过方法找到目标方法。

> 面向服务的架构(Service-Oriented Architecture)是一种组件模型，它通过将应用程序的不同功能单元(称为服务)进行拆分，并通过这些服务之间定义良好的接口和规范联系起来。接口是采用中立的方式来进行定义的，接口要独立于实现服务的硬件平台、操作系统、编程语言。这使得运行在各式各样服务器中的服务可以以一种统一和通用的方式交互。

## FastDFS

### 组件

* Tracker负责负载均衡和注册中心
  * 内部注册了很多Tracker

* Storage负责文件存储、文件管理
  * Storage内注册了很多group
  * 每个group内还有许多主机

### 工作流程

1. Storage通过心跳沟通定时注册到Tracker
2. 用户访问服务器上传文件Controller->/Post /upload
3. 服务器向Tracker查询可用的Stroage
4. Tracker返回可用的Storage
5. 程序根据返回的Storage找到可用的Storage，实现文件管理

### 组内和组间

* group内为文件同步 冗余备份，是一种容灾机制；一个group内的成员为同步节点，没有主从概念，不是集群
* 一堆group才能构成集群

### 访问路径

http://ip:port/group1/M00/00/00/lkjflLJFLKAJljlkjadlfk.jpeg

* 组名：文件上传后文件服务器所属的组
* 虚拟磁盘路径：storage配置的虚拟路径，与磁盘选项store_path对应
* 数据两级目录：用于存储数据，目录名是fastdfs计算产生
* 文件名：不同于上传时的文件名，而是由storage服务器根据文件上传时的信息生成

# 安装

启动docker

```shell
sudo systemctl start docker #手动启动
sudo systemctl enable docker.service #docker开机自启
```

搜索可用的fastdfs镜像

```shell
docker search fastdfs
```

pull下来要用的镜像

```shell
docker pull morunchang/fastdfs
```

run tracker容器

```shell
docker run -d --name tracker --net=host morunchang/fastdfs sh tracker.sh
```

* -d 后台运行
* --name docker容器的名字
* --net 设置网络模式

run storage容器

* -e 添加到环境变量
* TRACKER_IP 设置宿主机的ip
* GROUP_NAME 给storage设置一个组名

```shell
docker run -d --name storage --net=host -e TRACKER_IP='Centos的ip':22122 -e GROUP_NAME=group1 morunchang/fastdfs sh storage.sh
```

开放FastDFS端口

```shell
#storage
firewall-cmd --zone=public --add-port=23000/tcp --permanent
#tracker
firewall-cmd --zone=public --add-port=22122/tcp --permanent
#nginx
firewall-cmd --zone=public --add-port=8080/tcp --permanent 
#重启防火墙
firewall-cmd --reload
```

每次关机电脑都会stop容器

```shell
docker stop 容器ip或容器名 停止容器
docker ps -a 可以看到所有的容器
docker ps 可以看到运行中的容器
docker kill 容器ip或容器名 强行停止容器
docker restart 容器id或容器名 重启容器
```

设置容器自动启动

```shell
docker update --restart=always tracker
docker update --restart=always storage
```

进入storage容器，查看内置的nginx配置

```shell
docker exec -it storage /bin/bash
cd /etc/nginx/conf/
vim nginx.conf
```

配置文件如下：

```shell
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       8080;
        server_name  localhost;

        #charset koi8-r;
                #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        location ~ /M00 {
        			add_header Cache-Control no-store; #这句要自己加上，否则删除文件后，浏览器有本地缓存仍可访问数据
                    root /data/fast_data/data;
                    ngx_fastdfs_module;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
                # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }
    
    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;
    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}



```

