---
layout: post
title: Apache/httpd安装
excerpt: "httpd是Apache Http Server Program，它被设计为独立运行的守护进程。当启动后，它将创建子线程池或子进程池来处理请求。通常情况下，httpd不会被直接调用，在Unix系统上通过apachectl被调用，在Windows NT系统上通过一项服务被调用。<br/>由于PHP需要一个HTTP服务器程序来使用，所以在安装PHP前也需要安装httpd程序。"
date:   2020-09-26 09:30:00
categories: [WEB]
comments: true
---

## 安装流程

> 不要安装在Kali Linux系统下，Kali中已经有apache2了

1. 检查系统中是否已经有httpd程序在运行

   ```shell
   ps -ef | grep httpd
   #ps即Process Status，该指令会列出执行ps命令的那个时刻系统中执行的所有进程
   #grep即Global Regular Expression Print，它搜索指定文件的内容匹配特定的内容，默认情况下输出匹配成功所在的行，但是grep只支持匹配，但是不支持替换
   ```

2. 先检查是否已安装GCC/GCC-c++

   ```shell
   gcc -v
   #如果有返回信息，说明已经安装了GCC，若没有执行下面这条语句
   sudo apt-get install build-essential
   ```

3. 由于httpd安装需要依赖于APR、APR-Util、PCRE。将它们安装在/usr/local

   ```shell
   sudo mkdir /usr/local/httpd
   sudo mkdir /usr/local/apr
   sudo mkdir /usr/local/apr-util
   sudo mkdir /usr/local/pcre
   ```

4. 所以依次下载[APR、APR-Util](http://apr.apache.org/download.cgi)、[PCRE](https://sourceforge.net/projects/pcre/files/pcre/)、[Apache官方网站](http://httpd.apache.org/download.cgi)的安装包。

5. 解压压缩包

   ```shell
   tar -zxf *文件名*
   #-x 从存档中提取文件
   #-z 过滤文档gzip
   #-f 使用归档文件
   ```
   
6. 安装APR

   ```shell
   cd apr-1.7.0
   sudo ./configure --prefix=/usr/local/apr
   sudo make
   sudo make install
   ```

   * configure：配置。它是一个可执行脚本，用于探测操作系统的特性和相关依赖，根据检测结果生成Makefile文件。在待安装的源码路径下使用*./configure --help*输出详细功能。其中prefix是配置安装的路径，可以指定把所有资源文件放在/usr/local/xxx路径下，不会杂乱。否则会比较杂乱：
     * 可执行文件会在/usr/local/bin
     * 库文件会在/usr/local/lib
     * 配置文件在/usr/local/etc
     * 其他资源在/usr/local/share
   * make：编译。执行该命令需要从Makefile文件中读取信息。
   * make install：安装，它也从Makefile文件中读取信息
   * 每一步骤执行完可以通过*echo $?*查看命令是否执行成功，如果成功，返回0

7. 安装apr-util

   ```shell
   cd apr-util-1.6.1
   sudo ./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr/bin/apr-1-config
   sudo make
   sudo make install
   ```

   * 可能会遇到找不到expat.h的问题，原因是缺少expat。安装之后需要重新探测环境，执行**configure**。

     ```shell
     sudo apt-get install  libexpat1-dev	#Ubuntu
     sudo yum install -y expat-devel	#Centos
     ```

8. 安装pcre

   ```shell
   cd pcre-8.44
   sudo ./configure --prefix=/usr/local/pcre --with-apr=/usr/local/apr/bin/apr-1-config
   sudo make
   sudo make install
   ```

9. 安装httpd

   ```shell
   cd httpd-2.4.46
   sudo ./configure --prefix=/usr/local/httpd --with-pcre=/usr/local/pcre --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util
   sudo make
   sudo make install
   ```

10. 配置Apache

    ```shell
    cd /usr/local/httpd/conf
    vim httpd.conf
    ```

    * Listen *port* ，默认为80端口

11. 启动、关闭、重启

    ```shell
    cd /usr/local/httpd/bin
    ./apachectl start
    ./apachectl stop
    ./apachectl restart
    ```

    * httpd通常不会直接执行
    * 在Unix系统中，被apachectl调用

12. 由于每次到*/usr/local/httpd/bin/apachectl*执行启动比较麻烦，所以可以把它设置成Linux系统服务，并设置开机启动

    ```shell
    sudo cp /usr/local/httpd/bin/apachectl /etc/rc.d/init.d/httpd #centos
    sudo cp /usr/local/httpd/bin/apachectl /etc/init.d/httpd #ubuntu、kali
    
    #重启电脑之后，就可以通过service控制httpd了
    ```

    * runlevel control directory，大多数Linux发行版本中，启动脚本都被放在*/etc/rc.d/init.d/*

13. 常用命令

    ```shell
    sudo service httpd start	#启动
    sudo service httpd stop	#关闭
    sudo service httpd restart	#重启
    sudo systemctl status httpd.service	#查看状态
    ```

    