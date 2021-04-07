---
title: ping
toc: true
authors:
  - kevinwu
date: '2021-04-04'
lastmod: '2021-04-04'
draft: false
---

## 功能概述
ping基于ICMP协议，用于探测目标域名或者IP的连通性。

## 使用示例
* 示例一：ping www.baidu.com
```bash
kevinwu@debian:~$ ping www.baidu.com
PING www.a.shifen.com (220.181.38.150) 56(84) bytes of data.
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=1 ttl=128 time=59.0 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=2 ttl=128 time=57.5 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=3 ttl=128 time=55.2 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=4 ttl=128 time=62.5 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=5 ttl=128 time=78.1 ms
64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=6 ttl=128 time=56.7 ms
^C
--- www.a.shifen.com ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 781ms
rtt min/avg/max/mdev = 55.211/61.498/78.051/7.740 ms
```

界面说明：
1. 第一行"PING www.a.shifen.com (220.181.38.150) 56(84) bytes of data."表示目标域名的IP地址，发送数据的大小（带报文头部和不带报文头部）。
2. 接下来若干行"64 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=1 ttl=128 time=59.0 ms"表示请求序号，TTL信息（报文每经过一个路由器TTL减1），RTT（往返耗时）。
3. 分析报告第一行"6 packets transmitted, 6 received, 0% packet loss, time 781ms"表示发送包数量、接收包数量，丢包率。最后一个time字段有点意义不明，似乎应该显示ping持续的时长，但有BUG。具体详见：[https://github.com/iputils/iputils/issues/193](https://github.com/iputils/iputils/issues/193)。
4. 分析报告第二行"rtt min/avg/max/mdev = 55.211/61.498/78.051/7.740 ms"分别表示最小/平均/最大/硬件设备的耗时。

* 示例二：ping www.baidu.com，指定TTL=1
```bash
kevinwu@debian:~$ ping -t 1 www.baidu.com
PING www.a.shifen.com (220.181.38.150) 56(84) bytes of data.
From LFRZ_A1M2_202-08-11-47.TOR-1-127.40GE0-49 (172.16.38.2) icmp_seq=1 Time to live exceeded
From LFRZ_A1M2_202-08-11-47.TOR-1-127.40GE0-49 (172.16.38.2) icmp_seq=2 Time to live exceeded
From LFRZ_A1M2_202-08-11-47.TOR-1-127.40GE0-49 (172.16.38.2) icmp_seq=3 Time to live exceeded
From LFRZ_A1M2_202-08-11-47.TOR-1-127.40GE0-49 (172.16.38.2) icmp_seq=4 Time to live exceeded
From LFRZ_A1M2_202-08-11-47.TOR-1-127.40GE0-49 (172.16.38.2) icmp_seq=5 Time to live exceeded
^C
--- www.a.shifen.com ping statistics ---
5 packets transmitted, 0 received, +5 errors, 100% packet loss, time 58ms
```
可以看出，当TTL经过路由器减少到0之后，会被丢弃并产生ICMP错误报文"Time to live exceeded"

* 示例三：ping www.baidu.com，指定包大小为120字节
```bash
kevinwu@debian:~$ ping -s 120 www.baidu.com
PING www.a.shifen.com (220.181.38.150) 120(148) bytes of data.
128 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=1 ttl=128 time=58.3 ms
128 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=2 ttl=128 time=54.7 ms
128 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=3 ttl=128 time=89.9 ms
128 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=4 ttl=128 time=60.2 ms
128 bytes from 220.181.38.150 (220.181.38.150): icmp_seq=5 ttl=128 time=79.4 ms
^C
--- www.a.shifen.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 54.725/68.499/89.935/13.728 ms
```
