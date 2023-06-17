---
layout: post
title: Wireshark入门(三)-捕获过滤器
excerpt: "本文为阅读《Wirshark与Metasploit实战指南》的学习笔记。"
date:   2020-09-19 09:03:00
categories: [CyberSecurity]
comments: true
---

## 学习笔记

### 1、过滤器

过滤器用于快速减少显示分组，是从未分析会话中提取重要信息的最佳工具。Wireshark的过滤引擎可以缩小分组列表中展示的分组数目，使得通信流和网络设备的活动轨迹变得清晰起来。

Wireshark支持两种过滤器：展示过滤器(Display Filter)和捕获过滤器(Capture Filter)

UI位置：

* 捕获过滤器位于程序启动界面的接口列表之上
* 展示过滤器位于分组列表面板之上

配置时间：

* 捕获过滤器在捕获开始之前设置，在捕获过程中不允许修改
* 展示过滤器可以随时修改

提供功能：

* 捕获过滤器：在捕获数据包阶段发挥作用，将不符合规则的数据包丢弃，采用BPF(Berkeley Packet Filter)底层语法	
* 展示过滤器：只影响哪些分组在分组列表中进行显示，采用和常用编程语言类似的语法

使用场景：

* 如果希望在捕获和存储数据时就限制分组数量，则可以采用捕获筛选器

* 如果希望把所有分组保存下来，只展示重要分组，则可以采用展示筛选器

#### 捕获筛选器

> Wireshark对于捕获过滤器编写了一份简要的[官方文档](https://gitlab.com/wireshark/wireshark/-/wikis/CaptureFilters)
>
> 完整的BPF语法，可以查阅[BPF语法文档](http://www.tcpdump.org/manpages/pcap-filter.7.html)

**pcap_compile()**可以将一个字符串编译为一个捕获过滤器程序。

过滤器表达式包含一或多个基元。基元通常包含一个id、名字、数组，跟随着一或多个修饰符。

修饰符一共有三种：

* *type*：该修饰符用来表明id、数字属于什么类型，*type*有host、net、port、portrange。当*type*缺省时，默认为host。
* *dir*：该修饰符用于指定流量的传输方向，*dir*有src、dst、src or dst、src and dst、ra、ta、addr1、addr2、addr3、addr4。当*dir*缺省时，默认为src or dst。ra、ta、addr1、addr2、addr3、addr4仅在802.11无线局域网连接中可用。
* *proto*：该修饰符用于限定特定的协议，*proto*有ether、fddi、tr、wlan、ip、ip6、arp、rarp、decnet、tcp、udp

除了这些还有部分特殊的原语：

* gateway：匹配使用特定主机作网关的分组数据
* broadcast：只匹配广播broadcast，不匹配单播unicast的流量
* less：小于，后跟长度
* great：大于，后跟长度

逻辑操作符：

* and(&&)
* or(\|\|)
* not(!)

| 语法示例                 | 功能                                                         | 示例                                                         |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| host *host*              | 捕获流入和流出该ip的流量                                     | host 192.168.2.2                                             |
| net *net* mask *netmask* | 捕获流入和流出一个网段的流量(子网掩码形式)                   | net 192.168.0.0 mask 255.255.0.0                             |
| net xxx.xxx.xxx.xxx/xx   | 捕获流入和流出一个网段的流量(CIDR形式)                       | net 192.168.0.0/16                                           |
| src                      | 只捕获流出                                                   | src net 192.168.0.0/16                                       |
| dst                      | 只捕获流入                                                   | dst host 192.168.5.1                                         |
| port *port*              | 仅捕获流入或流出指定单一端口的流量<br/>常用于捕获具有特定端口的协议 | port 53 (dns流量)<br/>port 80 (http流量)<br/>port 25 (smtp流量)<br/>port ...... |
| tcp[*start* : *length*]  |                                                              |                                                              |
| ip                       | 仅捕获ip流量，最短的过滤器，可以摆脱arp、stp等用不到ip层的协议 | ip                                                           |
| broadcast                | 仅匹配广播数据，常与其他原语配合使用                         | ether broadcast                                              |

## 独立思考

### 1. Wireshark与Burp Suite相比有什么优势？

* Wireshark侧重于数据帧，Burp Suite侧重于请求和响应。

* Wireshark一到七层全解析，Burp Suite侧重于应用层。

* Wireshark是从网卡驱动层面抓包，Burp Suite是作为代理抓取所代理的包。

总之，Wireshark更擅长流量数据分析，分析网络协议等，无法改包和放包；Burp Suite更擅长渗透，可以轻松实现抓包、改包、放包等，但无法检测和分析底层协议。

### 2. Promiscuous mode 和 Monitor mode有什么区别？

混杂模式：在网卡**建立连接**的情况下，抓取网卡中能接收到的数据，而不论目标地址是不是它

监听模式：无线网卡特有的模式，允许在**不接入WiFi**的情况下，抓取当前频段的所有WiFi包

平时，也用混杂模式描述无线网卡的监听模式。

## 产生过的疑问

1. Wireshark与Burp Suite相比有什么优势？
2. promiscuous mode 和 monitor mode有什么区别？

