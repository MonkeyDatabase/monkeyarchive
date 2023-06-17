---
layout: post
title: Centos设置静态ip
excerpt: "在Vmware运行Centos虚拟机时，为了测试方便，需要给Centos设置静态ip。"
date:   2020-06-19 8:15:00
categories: [WEB]
comments: true
---



查看要修改的网卡名

```shell
ifconfig
```

进入网卡配置文件目录

```shell
cd /etc/sysconfig/network-scripts/
```

备份旧配置文件

```shell
cp ifcfg-xxxx /home/'username'/'yourbackupdir'/ifcfg-xxxx.backup
```

修改目标网卡配置文件，文件名格式为ifcfg-"网卡名"

```shell
vim ifcfg-xxxx
```

```c
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
IPADDR=192.168.211.130 	//修改为想要的固定ip
NETMASK=255.255.255.0	//设置新子网掩码
GATEWAY=192.168.211.2	//配置网关
DNS1=114.114.144.144	//配置DNS服务器
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=f0bc780c-6045-40fa-876f-fbb021366574
DEVICE=ens33
ONBOOT=yes				//系统启动时激活网卡
```

重启网卡

```shell
systemctl restart network 	//centos7
nmcli c reload				//centos8
```



