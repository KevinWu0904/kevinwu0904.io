---
title: 实时音视频业务中的高可用网络传输
toc: true
authors:
  - kevinwu
tags:
  - rtc
  - architecture
series:
  - 架构设计精粹
date: '2021-08-13'
lastmod: '2021-08-13'
draft: false
---

**RTC**（Real-Time Communication，实时通信）是音视频业务中难度最高、挑战最大的场景。不同于当下热门的直播技术，RTC对实时性的要求极高，连续渲染两个视频帧间隔在**500ms以上**即会对用户造成明显的卡顿感。

而在RTC中，**网络**是制约品质的**第一要素**。

## WebRTC
![WebRTC](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210729195344.png)

### 概述
什么是WebRTC？[**WebRTC**](https://webrtc.org/)的诞生和开源极大降低了实时通信的技术门槛，从而造就了一代又一代的音视频产品。2010年5月，Google以6820万美金收购了Global IP Solutions公司，从而拥有其**编解码**、**回声消除**、**P2P**等核心技术。Google随后于2011年5月开放了工程的源代码，并与IETF和W3C定制了行业标准，组成现在的WebRTC项目。

![WebRTC架构](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817152223.png)

WebRTC主要包括：
1. 标准的协议栈，主要包括**推荐**用于信令交互的TCP协议栈和用于媒体数据传输的UDP协议栈。
2. WebRTC C++实现和C++ Native API，可以用于开发者直接用在客户端App上面。
3. 标准的JavaScript API，主要用于给浏览器直接使用。

![WebRTC协议栈](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817152828.png)

WebRTC的设计初衷是**让两个浏览器之间具备直接实时音视频通信的能力**。因此一方面WebRTC中包含了音视频场景的核心技术，例如：编解码、回声消除、弱网对抗等；另一方面，WebRTC中还包含了完备的P2P技术，用来穿越企业数据中心网络中最常见的[**NAT**](https://en.wikipedia.org/wiki/Network_address_translation)。

![WebRTC设计初衷](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210730210236.png)

### 局限
WebRTC通过信令服务器交换通信双方的基本信息，并通过P2P技术在通信双方建立**直接的数据传输通道**（**PeerConnection**）。WebRTC非常适合于两点之间的数据传输，并且在此基础上包括编解码、弱网对抗、音频引擎、视频引擎等完备的音视频能力。

那么，如果是**三方通信**呢？

在三方通信的场景下，如果两两之间都能够互相通信，则需要两两之间建立PeerConnection，因此总体需要建立如下的“**三角关系**”：
![WebRTC 三方通话](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210810195833.png)

类似的，如果是**五方通信**呢？
![WebRTC 五方通话](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210810202106.png)

由此可见，随着参与通话的人数增多，基于WebRTC架构的通信模型复杂度将会**大大增加**，这样的full mesh结构显然是不具备可扩展性的。那么，WebRTC究竟是否能够用来做多方通话呢？

答案是**肯定的**，但需要借助**媒体服务器**来完成。

### 建连过程
音视频产品的服务器端通常会包括**信令**和**媒体**两个部分。其中，在客户端传输音视频数据之前，需要建立[ICE（Interactive Connectivity Establishment）](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment)连接，这部分网络协议有标准的RFC定义：[STUN](https://datatracker.ietf.org/doc/html/rfc5389)、[TURN](https://datatracker.ietf.org/doc/html/rfc5766)。并且需要借助STUN和TURN服务器来完成。让我们大致梳理一下通话双方的PeerConnection建立流程：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210813164143.png)

通信双方通过信令服务器来交换彼此的SDP信息。SDP是一份具有**特殊约定格式**的**纯文本**描述文档，其中包含了WebRTC建立连接所需要的ICE服务器信息、音视频编解码信息等。例如：
```
v=0
o=jdoe 2890844526 2890842807 IN IP4 10.47.16.5
s=SDP Seminar
i=A Seminar on the session description protocol
u=http://www.example.com/seminars/sdp.pdf
e=j.doe@example.com (Jane Doe)
c=IN IP4 224.2.17.12/127
t=2873397496 2873404696
a=recvonly
m=audio 49170 RTP/AVP 0
m=video 51372 RTP/AVP 99
a=rtpmap:99 h263-1998/90000
```
PeerConnection建立完成之后，就可以传输编码之后的音视频数据流了。因此，媒体服务器通常会同时实现STUN/TURN等协议的功能。

那么，媒体服务器的架构会是什么样子的呢？

## 企业级RTC架构
### MCU和SFU
由于WebRTC在多方通话上的局限，因此媒体服务器的架构是解决其扩展性问题的关键核心。目前主流的媒体服务器架构主要有两种类型：**MCU**和**SFU**。

MCU（**Multipoint Control Unit**）会使用中心化的节点与每个参会者保持一对一的连接。各发送方首先将自己的音视频流传输给MCU，再由MCU服务器将各路音视频流混合并转码，然后分别给每个参会者发送混合之后的音视频流。整体架构如下：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210816131009.png)

MCU在传统的多人会议中已经是比较成熟的架构，支持很多老旧设备。它也支持带宽自适应模式，因为混流单元可以为每个接收者生成不同质量的视频流。MCU可以充分利用硬件的解码能力，许多WebRTC客户端的芯片都具有解码单个视频流的能力。此外，由于MCU对于每个参会者来说，只有混合之后的音视频流，因此在网络传输上面比较节省带宽成本。

MCU的主要劣势是需要在服务端进行转码工作，会增大一定的延迟以及降低画质，同时混流会限制客户端的UI布局灵活性。

与此对应的另外一种架构是SFU（**Selective Forwarding Unit**），它会使用中心化的节点来直接转发参会者的音视频流。整体架构如下：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210816133329.png)

SFU服务端将不会参与任何转码工作，因此整体延迟小并且没有画质损失。通常SFU架构模式下，发送方需要支持生成一个流的多个版本（比如Simulcast或者VP8），SFU本身会根据客户端的带宽，动态调整发送不同清晰度的音视频流。

SFU的主要缺点是需要增加复杂的选流逻辑，对带宽的消耗也比较大。

不过总体来说，SFU的可扩展性比较强，是目前主流媒体服务器架构的首选。

### 级联传输
当参会双方所在的物理位置相隔较远的情况下，媒体服务器通常会选择在两地就近部署，然后通过媒体服务器来传输音视频数据，这样的方式也称为**级联传输**。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817152037.png)

### 跨国部署
跨国部署是企业保障异地接入体验所需要考虑的**进阶架构**。现代企业通常整体IT架构都是以**云**和**数据中心**为基石，RTC跨国部署结合数据中心，通常会形成**中心化的信令服务器**和**边缘化的媒体服务器**的架构。

![中美跨国部署架构](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817164554.png)

* 信令服务器和媒体服务器：媒体服务器与信令服务器之间需要保持心跳连接，信令服务器将会根据用户接入的IP地址判断用户所在地，然后就近选择合适的媒体服务器与用户交互。
* 核心数据中心和边缘数据中心：媒体服务器几乎没有外部依赖（例如：存储、网关等等）并且需要与用户直接通信，因此更加适合部署在**三线汇聚**（移动、联通、电信）的BGP机房，共享相同的公网IP；信令服务器对于机房建设要求相对较高一些，并且很有可能需要存储，因此更加适合部署在核心数据中心。

## 网络传输的高可用
首先需要声明的是WebRTC自身已经包含强大的弱网对抗算法，能够对抗一定程度的带宽限制、网络丢包、延时抖动等现象。但是无论算法**再如何强大**，网络环境差仍然是制约品质的第一要素。

因此从客户端到服务端之间，要保障高可用的网络传输，要分别从接入层、信令层、媒体层去思考。整体来说，保障高可用的思路主要有：
1. CDN：用户到数据中心之间的网络状况通常是**难以保障的**，CDN加速非常适合用来提升这样的场景。
2. 冗余和去重：信令是具有**时效性**和**不可丢失性**的特点，无论是丢失还是迟到，对于RTC业务场景来说都是致命打击，因此**冗余传输路径并且去重**是核心优化思路。
3. 专线和公网：企业专线的稳定性未必如想象中的那么高，如果能在此基础上引入**公网降级**，将会在极端场景下保障可用性。
4. SDN：软件定义网络，SDN的核心作用在于**选路**，通过SDN的实时监控和选路算法，让媒体级联传输更具智能。

### 接入层的高可用传输
接入层的高可用传输解决的是**从用户到网关之间的路径**。我们知道，用户和数据中心之间包括基站、运营商、路由器等等中间设备，网络拥塞问题很难解决，因此现代互联网网络拥塞也促进了CDN的兴起。

对于接入层的高可用传输，核心有两点：
1. CDN：CDN目的在于增强**用户触达**，用户与CDN节点距离相对比较近，CDN与核心数据中心之间的网络传输质量将会**远远高于**用户直接与核心数据中心通信。
2. 冗余去重：客户的网络通常会是移动、联通、电信三大运营商之一，但是不同的IP地址对不同的运营商表现可能不同，接入信令服务器的域名时候可以考虑同时接入三个域名，每个域名解析不同的线路，或者同时接入两个三线汇聚的域名。每次传输信令的时候都需要沿着多条链路发送，然后在接收端去除重复消息。

![接入层高可用传输](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817195516.png)

### 信令服务器的高可用传输
信令服务器的高可用传输解决的是**跨数据中心之间的路径**。对于通信方处于全球各地不同位置的场景，不同数据中心的信令服务器之间需要**交换数据**，此时跨数据传输的网络链路是非常关键的。

对于信令服务器的高可用传输，核心有两点：
1. 冗余去重：思路可以类比接入层的高可用。
2. 专线+公网：不同于接入层高可用，数据中心之间的网络通常来说会有企业专线的建设，用于保障企业内部的业务正常运作。因此信令通道可以同时利用专线和公网，公网主要用于降级。

![信令服务器高可用传输](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817212223.png)

### 媒体服务器的高可用传输
媒体服务器的高可用传输解决的是**媒体与媒体之间级联的路径**，并且对于媒体这种传输数据量极大的场景，不能采用冗余去重的思路去做，这对于带宽是**毁灭性打击**。因此，媒体服务器的最后一公里需要依赖SDN（软件定义网络）来帮助我们通过实时监控**智能选路**。

对于媒体服务器的高可用传输，核心点在于：
通过SDN控制面的监控和算法决定当前一段时间的媒体传输路径，媒体与媒体之间需要有级联传输，因此优秀的路径会减少网络的延迟和丢包。

![媒体服务器高可用传输](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-vc-network/20210817213723.png)

## 参考
1. [WebRTC介绍](https://github.com/bovinphang/WebRTC)
2. [WebRTC学习指南](https://webrtc.mthli.com/)

## 总结
本文总结了音视频RTC业务场景中最核心的网络高可用建设，通过冗余去重、公网降级和软件定义网络作为核心手段，可以最大化优化从用户到服务器再到用户之间的所有链路，再配合WebRTC的弱网对抗算法，整体达到优质的音视频体验！