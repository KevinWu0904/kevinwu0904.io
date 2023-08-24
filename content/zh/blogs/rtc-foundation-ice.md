---
title: 从零开始学习RTC——NAT穿透与ICE
toc: true
authors:
  - kevinwu
tags:
  - rtc
series:
  - 从零开始学习RTC
date: '2022-05-18'
lastmod: '2021-05-18'
draft: true
---

## 前言
在WebRTC中，浏览器之间的音视频传输是基于**点对点通信**完成的，即**RTCPeerConnection**。然而，现代网络设备因为IPv4的上限限制，通常会位于[NAT](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)之后。这类隐藏在NAT内部的网络地址也称为**内网地址**或者**私有地址**，与之对应的在互联网上面暴露的网络地址也称为**公网地址**或者**公有地址**。

NAT的出现为互联网IPv4地址资源耗尽做出了极大贡献，保障了近20年的互联网发展，为IPv6的过渡建立足够的时间基础。但与此同时，NAT也使得点对点通信难度更大。在RTC领域，尤其是WebRTC的模型中，如何穿透NAT是一个需要讨论的话题。

## NAT
我们知道，IPv4预定义了私有地址，用于构建企业内部的网络系统：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220518181207.png)

而NAT的作用正是将私有地址与公有地址做映射关系：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220520150800.png)

NAT主要以四种形式存在：
* 完全锥型
* 地址受限型
* 端口受限型
* 对称型

### 完全锥型
完全锥型的NAT将同一私有地址的<IP，端口>二元组**固定映射**到同一公有地址的<IP，端口>二元组，且任何外网主机均可以通过访问映射之后的公有地址，实现访问内部主机设备的功能。

该类型的NAT在映射之后，对外不做限制可以双向访问，因此称为**完全锥型**。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220520154329.png)

完全锥型的NAT在安全性方面不够完善，因此实际情况下会对地址或者端口进一步做限制。

### 地址受限型
地址受限型的NAT在完全锥型的基础上增加了**IP的限制**，只有内网主机主动连接过的外部IP，才可以通过NAT向内部通信。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220520160134.png)

### 端口受限型
端口受限型的NAT在地址受限型的基础上更进一步增加了**端口的限制**，只有内网主机主动连接过的外部<IP，端口>二元组，才可以通过NAT向内部通信。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220520160312.png)

### 对称型
对称型的NAT最为严格，同一私有地址的<IP，端口>访问外部主机的<IP，端口>，这一个四元组<源IP，源端口，目标IP，目标端口>中任何一个元素发生变化，都会对公网地址的<IP，端口>产生影响。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220520161809.png)

## ICE
ICE(Interactive Connectivity Establishment，交互性连通性建立)实际上是一套允许点对点通信彼此发现并且建立连接过程的框架。关于ICE，我们需要知道：

1. ICE寻找媒体传输的最短路径：ICE能够收集所有本地的IP+端口候选者，在通信两端的ICE候选者之间探测并最终选择合适的候选者对建立连接。
2. ICE允许穿透NAT和防火墙：ICE借助STUN协议和TURN协议，能够增加穿透防火墙和NAT的几率。这些协议会找到UDP端口和TCP端口，允许对等端发送和接收媒体。

### STUN
STUN（Session Traversal Utilities for NAT）的作用是让内网客户端感知到经过NAT映射之后的公网地址。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220520165023.png)

STUN server可以免费获取，比如Google提供的`stun:stun.l.google.com:19302`。

### TURN
TURN（Traversal Using Relays around NAT）在STUN协议基础上，新增中继功能，针对无法直接完成P2P通信的NAT类型。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-rtc-foundation-ice/20220525112529.png)

### ICE的详细过程
ICE建立连接的过程如下：
1. 地址收集（Gathering Candidates）


2. 交换SDP
3. 连通性检测（Connectivity Checks）
4. 确定可用通信链路（Nominating Candidate Pairs）
5. ICE状态确定


## 参考
* [NAT的四种分类](https://blog.csdn.net/s2603898260/article/details/118755474)
* [STUN](https://datatracker.ietf.org/doc/rfc5389/)
* [ICE](https://datatracker.ietf.org/doc/rfc8445/)