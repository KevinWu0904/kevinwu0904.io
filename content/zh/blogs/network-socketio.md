---
title: 【译】Socket.IO协议
toc: true
authors:
  - kevinwu
tags:
  - network
series:
  - 网络协议
date: '2021-04-28'
lastmod: '2021-04-28'
draft: false
---

## 前言
[**Socket.IO**](https://github.com/socketio/socket.io)是一个优秀的JavaScript实时通信框架。它提供基于事件的双向流通信模型，底层封装了Websocket和HTTP long-polling。

Socket.IO的设计基于它定义的标准协议，本质上是语言无关的。因此除了官方提供的NodeJS版本之外，也有其它语言的开源实现，例如：[**go-socket.io**](https://github.com/googollee/go-socket.io)。

笔者在工作中曾基于go-socket.io完成音视频信令服务器的相关功能，对框架的优秀能力有比较深刻的直观感受。因此本文通过对Socket.IO协议的翻译，旨在进一步理解Socket.IO的底层实现原理。

另一点比较遗憾的是，go-socket.io支持的协议版本远落后于官方的NodeJS实现，并且该开源项目暂时没有太大进展。因此笔者也希望通过加深对新版本协议的理解，来尝试进一步完善go-socket.io。

## 协议原文
协议原文的地址：[socket.io-protocol](https://github.com/socketio/socket.io-protocol)

## 协议译文

### Socket.IO 协议
本文描述了Socket.IO协议。可参考基于JavaScript语言的具体实现：[socket.io-parser](https://github.com/socketio/socket.io-parser)，[socket.io-client](https://github.com/socketio/socket.io-client)和[socket.io](https://github.com/socketio/socket.io)。

### 协议版本
这是Socket.IO协议的第5个版本，在`socket.io@3.0.0...latest`中已经实现。

第4个版本详见：[https://github.com/socketio/socket.io-protocol/tree/v4](https://github.com/socketio/socket.io-protocol/tree/v4)

第3个版本详见：[https://github.com/socketio/socket.io-protocol/tree/v3](https://github.com/socketio/socket.io-protocol/tree/v3)

第1个版本和第2个版本的协议包括在Socket.IO 1.0的工作内容当中，但并没有发布过正式的Release协议版本。

本协议是构建在[Engine.IO协议](https://github.com/socketio/engine.io-protocol)的第4个版本基础之上。

Engine.IO描述的是一个基于Websocket和HTTP long-polling更底层的基础传输系统，而Socket.IO则是在它之上的一层封装。Socket.IO支持如下的功能：

* 多路复用（名称空间）

JavaScript API示例：
```javascript
// server-side
const nsp = io.of("/admin");
nsp.on("connect", socket => {});
// client-side
const socket1 = io(); // main namespace
const socket2 = io("/admin");
socket2.on("connect", () => {});
```

* Packet的ACK

JavaScript API示例：
```javascript
// on one side
socket.emit("hello", 1, () => { console.log("received"); });
// on the other side
socket.on("hello", (a, cb) => { cb(); });
```

### Packet格式
一个Packet包括以下字段：
* 类型（整数，详见下文）
* 名称空间（字符串）
* 可选字段：Packet内容（对象 | 数组）
* 可选字段：ACK的ID（整数）

### Packet类型
#### 0-CONNECT
该事件发生时机：
* 当客户端连接一个名称空间。客户端会发送一个用于鉴权的payload，例如：
```json
{
  "type": 0,
  "nsp": "/admin",
  "data": {
    "token": "123"
  }
}
```

* 当服务端接受来自一个名称空间的连接。请求成功后，服务端会响应一个带有Socket ID的payload，例如：
```json
{
  "type": 0,
  "nsp": "/admin",
  "data": {
    "sid": "CjdVH4TQvovi1VvgAC5Z"
  }
}
```

#### 1-DISCONNECT
该事件发生在一端想要断开名称空间的连接时。它不包括任何payload和ACK ID，例如：
```json
{
  "type": 1,
  "nsp": "/admin"
}
```

#### 2-EVENT
该事件发生在一端想要给另一端传输数据（不包括二进制数据）时。它包括payload，可能还会包括ACK ID，例如：
```json
{
  "type": 2,
  "nsp": "/",
  "data": ["hello", 1]
}
```

包括ACK ID的样例：
```json
{
  "type": 2,
  "nsp": "/admin",
  "data": ["project:delete", 123],
  "id": 456
}
```

#### 3-ACK
该事件发生在一端接收到EVENT事件或带有ACK ID的BINARY_EVENT事件。它包括对之前这个事件的ACK ID，可能还会包括payload（不包括二进制数据），例如：
```json
{
  "type": 3,
  "nsp": "/admin",
  "data": [],
  "id": 456
}
```

#### 4-CONNECT_ERROR
该事件发生在服务端拒绝一个名称空间的连接时。它包括一个"message"字段，可能还会包括一个"data"字段，例如：
```json
{
  "type": 4,
  "nsp": "/admin",
  "data": {
    "message": "Not authorized",
    "data": {
      "code": "E001",
      "label": "Invalid credentials"
    }
  }
}
```

#### 5-BINARY_EVENT
注意：BINARY_EVENT和BINARY_ACK都用于内建的解析器中，为了区别出包中是否包括二进制内容。它们不会用于其他自定义解析器中。

该事件发生在一端想要给另一端传输数据（包括二进制数据）时。它包括payload，可能还会包括ACK ID，例如：
```json
{
  "type": 5,
  "nsp": "/",
  "data": ["hello", <Buffer 01 02 03>]
}
```

包括ACK ID的样例：
```json
{
  "type": 5,
  "nsp": "/admin",
  "data": ["project:delete", <Buffer 01 02 03>],
  "id": 456
}
```

#### 6-BINARY_ACK
该事件发生在一端接收到EVENT事件或带有ACK ID的BINARY_EVENT事件。它包括对之前这个事件的ACK ID，可能还会包括payload（包括二进制数据），例如：
```json
{
  "type": 6,
  "nsp": "/admin",
  "data": [<Buffer 03 02 01>],
  "id": 456
}
```

### Packet编码
本小节描述了Socket.IO 客户端和服务端之间默认解析器的编码细节，源码实现参考：[这里](https://github.com/socketio/socket.io-parser)。

JavaScript服务端和客户端也支持自定义解析器，适用于不同场景。具体可以参考：[socket.io-json-parser](https://github.com/darrachequesne/socket.io-json-parser)和[socket.io-msgpack-parser](https://github.com/darrachequesne/socket.io-msgpack-parser)。

另外注意一点：Socket.IO的packet本质上是Engine.IO`message`类型的packet（关于Engine.IO参考[这里](https://github.com/socketio/engine.io-protocol)），所以编码结果发送时候会带上`4`这个数字前缀。（在HTTP long-polling的请求和响应包体中，或者在Websocket的数据帧中。）

#### 编码格式
```
<packet type>[<# of binary attachments>-][<namespace>,][<acknowledgment id>][JSON-stringified payload without binary]

+ binary attachments extracted
```

注意：
* 当名称空间不是默认的`/`时候才会出现在编码格式中。

#### 编码样例
* `/`名称空间`CONNECT`
```json
{
  "type": 0,
  "nsp": "/",
  "data": {
    "token": "123"
  }
}
```
编码为：`0{"token":"123"}`

* `/admin`名称空间的`CONNECT`
```json
{
  "type": 0,
  "nsp": "/admin",
  "data": {
    "token": "123"
  }
}
```json
编码为`0/admin,{"token":"123"}`

* `/admin`名称空间的`DISCONNECT`
```json
{
  "type": 1,
  "nsp": "/admin"
}
```
编码为`1/admin`

* `EVENT`
```json
{
  "type": 2,
  "nsp": "/",
  "data": ["hello", 1]
}
```
编码为`2["hello",1]`

* 带ACK ID的`EVENT`
```json
{
  "type": 2,
  "nsp": "/admin",
  "data": ["project:delete", 123],
  "id": 456
}
```
编码为`2/admin,456["project:delete",123]`

* `ACK`
```json
{
  "type": 3,
  "nsp": "/admin",
  "data": [],
  "id": 456
}
```
编码为`3/admin,456[]`

* `CONNECT_ERROR`
```json
{
  "type": 4,
  "nsp": "/admin",
  "data": {
    "message": "Not authorized"
  }
}
```
编码为`4/admin,{"message":"Not authorized"}`

* `BINARY_EVENT`
```json
{
  "type": 5,
  "nsp": "/",
  "data": ["hello", <Buffer 01 02 03>]
}
```
编码为`51-["hello",{"_placeholder":true,"num":0}] + <Buffer 01 02 03>`

* 带ACK ID的`BINARY_EVENT`
```json
{
  "type": 5,
  "nsp": "/admin",
  "data": ["project:delete", <Buffer 01 02 03>],
  "id": 456
}
```
编码为`51-/admin,456["project:delete",{"_placeholder":true,"num":0}] + <Buffer 01 02 03>`

* `BINARY_ACK`
```json
{
  "type": 6,
  "nsp": "/admin",
  "data": [<Buffer 03 02 01>],
  "id": 456
}
```
编码为`61-/admin,456[{"_placeholder":true,"num":0}] + <Buffer 03 02 01>`

### 协议交互
#### 连接名称空间
对于每个名称空间（包括主名称空间），客户端首先发送一个`CONNECT`，服务端回复一个带有Socket ID的`CONNECT`。
```
Client > { type: CONNECT, nsp: "/admin" }
Server > { type: CONNECT, nsp: "/admin", data: { sid: "wZX3oN0bSVIhsaknAAAI" } } (if the connection is successful)
or
Server > { type: CONNECT_ERROR, nsp: "/admin", data: { message: "Not authorized" } }
```

#### 名称空间断连
```
Client > { type: DISCONNECT, nsp: "/admin" }
```

反之亦然。同时另外一端无需任何消息回复。

#### ACK
```
Client > { type: EVENT, nsp: "/admin", data: ["hello"], id: 456 }
Server > { type: ACK, nsp: "/admin", data: [], id: 456 }
or
Server > { type: BINARY_ACK, nsp: "/admin", data: [ <Buffer 01 02 03> ], id: 456 }
```

反之亦然。

### 会话示例
这里有一份包括了Socket.IO和Engine.IO的会话示例。

* Request n°1 (open packet)
```
GET /socket.io/?EIO=4&transport=polling&t=N8hyd6w
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
0{"sid":"lv_VI97HAXpY6yYWAAAC","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":5000}
```

细节：
```
0           => Engine.IO "open" packet type
{"sid":...  => the Engine.IO handshake data
```

注意：参数`t`用于防止浏览器缓存该请求。

* Request n°2 (namespace connection request)
```
POST /socket.io/?EIO=4&transport=polling&t=N8hyd7H&sid=lv_VI97HAXpY6yYWAAAC
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
40
```

细节：
```
4           => Engine.IO "message" packet type
0           => Socket.IO "CONNECT" packet type
```

* Request n°3 (namespace connection approval)
```
GET /socket.io/?EIO=4&transport=polling&t=N8hyd7H&sid=lv_VI97HAXpY6yYWAAAC
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
40{"sid":"wZX3oN0bSVIhsaknAAAI"}
```

* Request n°4

服务端会执行`socket.emit('hey', 'Jude')`

```
GET /socket.io/?EIO=4&transport=polling&t=N8hyd7H&sid=lv_VI97HAXpY6yYWAAAC
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
42["hey","Jude"]
```

细节：
```
4           => Engine.IO "message" packet type
2           => Socket.IO "EVENT" packet type
[...]       => content
```

* Request n°5 (message out)

客户端会执行`socket.emit('hello'); socket.emit('world');`

```
POST /socket.io/?EIO=4&transport=polling&t=N8hzxke&sid=lv_VI97HAXpY6yYWAAAC
> Content-Type: text/plain; charset=UTF-8
42["hello"]\x1e42["world"]
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
ok
```

细节：
```
4           => Engine.IO "message" packet type
2           => Socket.IO "EVENT" packet type
["hello"]   => the 1st content
\x1e        => separator
4           => Engine.IO "message" packet type
2           => Socket.IO "EVENT" packet type
["world"]   => the 2nd content
```

* Request n°6 (WebSocket upgrade)
```
GET /socket.io/?EIO=4&transport=websocket&sid=lv_VI97HAXpY6yYWAAAC
< HTTP/1.1 101 Switching Protocols
```

WebSocket数据帧：
```
< 2probe                                        => Engine.IO probe request
> 3probe                                        => Engine.IO probe response
> 5                                             => Engine.IO "upgrade" packet type
> 42["hello"]
> 42["world"]
> 40/admin,                                     => request access to the admin namespace (Socket.IO "CONNECT" packet)
< 40/admin,{"sid":"-G5j-67EZFp-q59rADQM"}       => grant access to the admin namespace
> 42/admin,1["tellme"]                          => Socket.IO "EVENT" packet with acknowledgement
< 461-/admin,1[{"_placeholder":true,"num":0}]   => Socket.IO "BINARY_ACK" packet with a placeholder
< <binary>                                      => the binary attachment (sent in the following frame)
... after a while without message
> 2                                             => Engine.IO "ping" packet type
< 3                                             => Engine.IO "pong" packet type
> 1                                             => Engine.IO "close" packet type
```

### 版本历史
#### v5和v4的差异
* 移除默认名称空间的连接

在之前的版本中，客户端总是会跟默认名称空间建连，即使是另一个名称空间的请求。现在客户端必须发送一个`CONNECT`。

git commit：[09b6f23](https://github.com/socketio/socket.io/commit/09b6f2333950b8afc8c1400b504b01ad757876bd)（服务端）和 [249e0be](https://github.com/socketio/socket.io-client/commit/249e0bef9071e7afd785485961c4eef0094254e8)（客户端）

* `ERROR`重命名为`CONNECT_ERROR`

语义和数值`4`并没有改变：该packet类型依然是被用在服务端拒绝一个名称空间的连接上面。只是我们认为新的名称更合适。

git commit：[d16c035](https://github.com/socketio/socket.io/commit/d16c035d258b8deb138f71801cb5aeedcdb3f002) (服务端) 和 [13e1db7c](https://github.com/socketio/socket.io-client/commit/13e1db7c94291c583d843beaa9e06ee041ae4f26) (客户端)。

* `CONNECT`现在包含payload

客户端可以发送一个包含授权目的的payload。例如：
```json
{
  "type": 0,
  "nsp": "/admin",
  "data": {
    "token": "123"
  }
}
```

假设结果成功，那么服务端会返回包含Socket ID的payload。例如：
```json
{
  "type": 0,
  "nsp": "/admin",
  "data": {
    "sid": "CjdVH4TQvovi1VvgAC5Z"
  }
}
```

这个变更意味着Socket.IO的连接ID现在和底层Engine.IO的连接ID不是同一个（可以在HTTP请求的query参数中看到）。

git commit：[2875d2c](https://github.com/socketio/socket.io/commit/2875d2cfdfa463e64cb520099749f543bbc4eb15) (服务端) 和 [bbe94ad](https://github.com/socketio/socket.io-client/commit/bbe94adb822a306c6272e977d394e3e203cae25d) (客户端)

* `CONNECT_ERROR`现在是一个对象，而不再是一个原生字符串。

git commit：[54bf4a4](https://github.com/socketio/socket.io/commit/54bf4a44e9e896dfb64764ee7bd4e8823eb7dc7b) (服务单) 和 [0939395](https://github.com/socketio/socket.io-client/commit/09393952e3397a0c71f239ea983f8ec1623b7c21) (客户端)

#### v4和v3的差异
* 新增`BINARY_ACK`

过去的版本中，`ACK`总要假设它包括二进制数据，对于性能上面有一些损失。

#### v3和v2的差异
* 移除msgpack编码二进制数据包的功能。（参考[299849b](https://github.com/socketio/socket.io-parser/commit/299849b00294c3bc95817572441f3aca8ffb1f65)）

#### v2和v1的差异
* 新增`BINARY_EVENT`

这是在Socket.IO 1.0版本的实现中为了支持二进制数据引入的。`BINARY_EVENT`是由[msgpack](https://msgpack.org/)

#### 最初版本
最初的版本将Socket.IO和Engine.IO分离开。它虽然没有被某个release的Socket.IO框架实现，但给后面的迭代铺平了道路。

### 版权
MIT

## 参考
1. [RFC6455: The Websocket Protocol](https://tools.ietf.org/html/rfc6455)

## 总结
本文是对Socket.IO协议原文的译文，旨在深入学习该框架的实现原理。
