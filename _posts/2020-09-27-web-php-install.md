---
layout: post
title: 基于httpd的PHP7安装
excerpt: "在新搭好的服务器上部署PHP文件，启动服务器后PHP代码被注释，而且被传输到客户端，"
date:   2020-09-27 22:24:00
categories: [WEB]
comments: true
---

## 安装过程

1. 新建文件夹用于存放安装后的PHP

   ```shell
   sudo mkdir /usr/local/php
   ```

2. 下载[PHP](https://www.php.net/downloads)的安装包

3. 解压压缩包

   ```shell
   tar -zxf php-7.4.10.tar.gz 
   ```

4. 安装PHP

   ```shell
   cd php-7.4.10/
   sudo ./configure --prefix=/usr/local/php --with-apxs2=/usr/local/httpd/bin/apxs --with-mysql --with-kerberos --with-curl
   sudo make
   sudo make install
   ```

   * 遇到报错需要libxml-2.0，这些需要的是开发库，不是平时的运行库

     ```shell
     sudo apt-get install libxml2-dev
     ```

   * 遇到报错需要sqlite3

     ```shell
     sudo apt-get install libsqlite3-dev
     ```

   * 遇到报错需要libcurl

     ```shell
     sudo apt-get install libcurl4-openssl-dev
     ```

   * **注意**这些开发库安装之后，仍需要重新configure

5. 设置php.ini

   ```shell
   sudo cp php.ini-development  /usr/local/php/lib/php.ini
   ```

6. 配置apache/httpd的httpd.conf，使之加载php模块

   ```shell
   sudo vim /usr/local/httpd/conf/httpd.conf
   
   #在模块处添加下面一行，使之支持php7，很有可能在第4步的make install中脚本已经自动添加，但是建议手动检查一遍
   LoadModule php7_module modules/libphp7.so
   ```

7. 配置apache/httpd的httpd.conf，使之将php文件交给PHP模块处理。使用下面这种方法而不是直接使用Apache AddType，可以避免潜在的文件上传漏洞，比如*exploit.php.jpg*作为PHP执行，使用该方法，只需添加扩展名就可以将文件解析为PHP。

   ```xml
   <FilesMatch \.php$>
       SetHandler application/x-httpd-php
   </FilesMatch>
   ```

8. 重启httpd

   ```shell
   sudo service httpd restart
   ```

9. 测试PHP安装情况，默认的网站页面在/usr/local/httpd/htdocs

   ```shell
   cd /usr/local/httpd/htdocs
   sudo touch test.php
   sudo vim test.php
   
   #以下为添加的文件内容
   <?php
   	echo phpinfo();
   	?>
   #文件结束
   
   sudo service httpd restart
   
   #通过浏览器访问127.0.0.1/test.php,如果返回一个PHP的页面，内容为PHP相关的表格，则为安装完成
   ```

   
