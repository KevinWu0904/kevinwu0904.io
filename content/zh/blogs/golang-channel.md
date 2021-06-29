---
title: Go源码学习之channel
toc: true
authors:
  - kevinwu
tags:
  - golang
series:
  - 深入理解Go语言
date: '2021-06-29'
lastmod: '2021-06-29'
draft: false
---

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-common/golang-logo.png)

Go语言的并发模型基于[CSP（Communicating Sequential Processes）](https://spinroot.com/courses/summer/Papers/hoare_1978.pdf?__cf_chl_managed_tk__=b9821fe2bc6d612a6fe6cc26b2b29a3b16ac499c-1624872512-0-AfRgdbaTY6pF1ABILizpaNiE0bTQCqJEhWCpCvVWEHyHrLx18aOLKdHXddkkcmpLCJh1hZx1rA62608KoTkykcpqjTzuxbE76_wDnQDvT0wLpjWG9NvukUIsEZVYtrCx0-R8yy1JKXwYjyX4sYMKMbcJwJrdEF1UUMZc95A9SXi2epg5cYFsvcInWZOrnrRK4I78KjntipUkSuA-2eyOr4Nb3trGhs2EDIIBhmr9totXZXS8infibvBtZ8mbc4UpPIQfPLBjqGn-rUK_c7ZJlihAawuo6LYyl2s7giYyO3dsNNsgilsWLYC_77P7pyH9LX87GHQk7Z54VQbJ8CacqHNSxF2DSIkcBi7frxrw7v7Fj4-_hdIKlbkuh-sdXXymp8wmVWr4I6NrurcUBb3bj4NLNL61HDxwse3myGuFrWYHEn-3uK6RzdKmCJWa5B9Ou7HK3aN3fzK-p2GRJni9ppfdRXbIQ6o0zPvutyMIfh8QeggPkwxreUw2nok0ZNKxj1c8kMXhv3I5i0P3XAUqwg7ho-9X-kFr909xOzlnk69wz931O5KqYOu3eZdx9eRJiiqMUbkLGtxGE5EH_gOlQfjI1Xa2FzueZBqJk8HuHB53fd3iZnOmvpTpUvZXZZb1vu70gMzNxuhlSzYb2KU9gSE)理论。Go的并发哲学强调：
> *"Do not communicate by sharing memory; instead, share memory by communicating."*

**goroutine**和**channel**是Go语言CSP并发模型的两大核心概念，本文将从源码层面对channel进行深度解析。

![goroutine和channel](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210628184427.png)

## 版本说明
本文涉及到的源码解析部分来源于go官方的[**1.16.5**](https://github.com/golang/go/tree/go1.16.5)版本，其中英文部分的注释也属于源码，中文部分的注释则属于笔者添加。

## channel的定义
channel是Go语言的内置数据类型，以`chan`关键字作为Go语言的语法之一。要从源码角度解析channel，我们需要查看channel在runtime层面的定义：

```go
// go/src/runtime/chan.go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
channel的结构主要由以下几个部分组成：
1. **环形队列缓存**：当channel缓冲区**未满且仍有send请求**的时候，该队列将用于缓存此时收到的消息。
2. **双向链表**：当channel缓冲区**已满且仍有send请求**或者**已空但仍有recv请求**的时候，channel将进入阻塞状态。通过该双向链表存放阻塞的Goroutine信息，以**sudog**的形式包裹**G**。
3. **锁**：go语言的channel基于**悲观锁**实现。

![Channel结构](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210628210950.png)

如图所示，channel结构体的含义如下：
* `qcount`：环形队列缓存的实际大小，即当前缓存的消息个数。
* `dataqsiz`：环形队列缓存的容量，即允许缓存的消息最大个数。
* `buf`：环形队列缓存的具体数据。
* `sendx`：环形队列缓存的指针之一。
* `recvx`：环形队列缓存的指针之一。
* `sendq`：当缓冲区已满且仍有send请求，send方将会阻塞，该双向链表存放阻塞sudog。
* `recvq`：当缓冲区已空且仍有recv请求，recv方将会阻塞，该双向链表存放阻塞sudog。
* `closed`：是否已经关闭。
* `lock`：悲观锁。

## channel的创建
我们可以通过`make`函数创建`chan`，分为不带缓冲和带缓冲两种：
```go
package main

import "fmt"

func main() {
	ch1 := make(chan struct{})     // 不带缓冲
	ch2 := make(chan struct{}, 10) // 带缓冲
	fmt.Println(len(ch1), len(ch2))
}
```

不带缓冲和带缓冲的本质区别其实就在于**底层的环形队列缓存**。对于带缓冲的情况而言，消息可以被暂时存放进队列缓存中，**不会导致发送方的G直接进入阻塞状态**。

让我们通过go的官方工具来查看汇编结果：
```bash
$ go tool compile -S main.go

"".main STEXT size=310 args=0x0 locals=0x80 funcid=0x0
	0x0000 00000 (main.go:5)	TEXT	"".main(SB), ABIInternal, $128-0
	0x0000 00000 (main.go:5)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:5)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:5)	PCDATA	$0, $-2
	0x000d 00013 (main.go:5)	JLS	300
	0x0013 00019 (main.go:5)	PCDATA	$0, $-1
	0x0013 00019 (main.go:5)	ADDQ	$-128, SP
	0x0017 00023 (main.go:5)	MOVQ	BP, 120(SP)
	0x001c 00028 (main.go:5)	LEAQ	120(SP), BP
	0x0021 00033 (main.go:5)	FUNCDATA	$0, gclocals·7d2d5fca80364273fb07d5820a76fef4(SB)
	0x0021 00033 (main.go:5)	FUNCDATA	$1, gclocals·729b7dbc411005893d6fd941e5c100f6(SB)
	0x0021 00033 (main.go:5)	FUNCDATA	$2, "".main.stkobj(SB)
	0x0021 00033 (main.go:6)	LEAQ	type.chan struct {}(SB), AX
	0x0028 00040 (main.go:6)	MOVQ	AX, (SP)
	0x002c 00044 (main.go:6)	MOVQ	$0, 8(SP)
	0x0035 00053 (main.go:6)	PCDATA	$1, $0
	0x0035 00053 (main.go:6)	CALL	runtime.makechan(SB) # 创建不带缓冲的channel
	0x003a 00058 (main.go:6)	MOVQ	16(SP), AX
	0x003f 00063 (main.go:6)	MOVQ	AX, "".ch1+72(SP)
	0x0044 00068 (main.go:7)	LEAQ	type.chan struct {}(SB), CX
	0x004b 00075 (main.go:7)	MOVQ	CX, (SP)
	0x004f 00079 (main.go:7)	MOVQ	$10, 8(SP)
	0x0058 00088 (main.go:7)	PCDATA	$1, $1
	0x0058 00088 (main.go:7)	CALL	runtime.makechan(SB) # 创建带缓冲的channel
	0x005d 00093 (main.go:7)	MOVQ	16(SP), AX
	0x0062 00098 (main.go:8)	MOVQ	"".ch1+72(SP), CX
	0x0067 00103 (main.go:8)	TESTQ	CX, CX
	0x006a 00106 (main.go:8)	JEQ	293
	0x0070 00112 (main.go:8)	MOVQ	(CX), CX
	0x0073 00115 (main.go:8)	TESTQ	AX, AX
	0x0076 00118 (main.go:8)	JEQ	281
	0x007c 00124 (main.go:8)	MOVQ	(AX), AX
	0x007f 00127 (main.go:8)	MOVQ	AX, ""..autotmp_26+64(SP)
	0x0084 00132 (main.go:8)	MOVQ	CX, (SP)
	0x0088 00136 (main.go:8)	PCDATA	$1, $0
	0x0088 00136 (main.go:8)	CALL	runtime.convT64(SB)
	0x008d 00141 (main.go:8)	MOVQ	8(SP), AX
	0x0092 00146 (main.go:8)	MOVQ	AX, ""..autotmp_27+80(SP)
	0x0097 00151 (main.go:8)	MOVQ	""..autotmp_26+64(SP), CX
	0x009c 00156 (main.go:8)	MOVQ	CX, (SP)
	0x00a0 00160 (main.go:8)	PCDATA	$1, $2
	0x00a0 00160 (main.go:8)	CALL	runtime.convT64(SB)
	0x00a5 00165 (main.go:8)	MOVQ	8(SP), AX
	0x00aa 00170 (main.go:8)	XORPS	X0, X0
	0x00ad 00173 (main.go:8)	MOVUPS	X0, ""..autotmp_18+88(SP)
	0x00b2 00178 (main.go:8)	MOVUPS	X0, ""..autotmp_18+104(SP)
	0x00b7 00183 (main.go:8)	LEAQ	type.int(SB), CX
	0x00be 00190 (main.go:8)	MOVQ	CX, ""..autotmp_18+88(SP)
	0x00c3 00195 (main.go:8)	MOVQ	""..autotmp_27+80(SP), DX
	0x00c8 00200 (main.go:8)	MOVQ	DX, ""..autotmp_18+96(SP)
	0x00cd 00205 (main.go:8)	MOVQ	CX, ""..autotmp_18+104(SP)
	0x00d2 00210 (main.go:8)	MOVQ	AX, ""..autotmp_18+112(SP)
  ... 省略不相关的部分
```
从汇编结果来看，无论是带缓冲还是不带缓冲，channel的创建底层实现都是基于`runtime.makechan`函数。让我们进一步翻阅相关源码：
```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe. 
  // 算术溢出判断
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
  // 根据不同情况来分配内存：
  //     1. 不带缓冲：只给hchan分配内存
  //     2. 带缓冲且不包括指针类型：给hchan和环形队列缓存分配一段连续的内存空间
  //     3. 带缓冲且包括指针类型：给hchan和环形队列缓存分别分配内存空间
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

  // 更新以下字段的值
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```
`runtime.makechan`为新创建的channel分配内存空间，考虑三种情况：

1. 不带缓冲区：只需要给`hchan`分配内存空间。
2. 带缓冲区且不包括指针类型：同时给`hchan`和环形队列缓存`buf`分配一段连续的内存空间。
3. 带缓冲区且包括指针类型：分别给`hchan`和环形队列缓存`buf`分配不同的内存空间。

## channel的发送和接收

### 基本原则
channel的核心作用在于为不同的goroutine传输数据，因此首先要声明强调的是：channel的发送和接收必须位于**不同的**goroutine内，在相同的goroutine内同时给channel发送和接收数据则会引发**死锁**问题：
```go
package main

func main() {
	ch := make(chan struct{})

	ch <- struct{}{}

	<-ch
}
```

```bash
$ go run main.go

fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /root/go/src/gogogo/main.go:6 +0x53
exit status 2
```

### 三种情况
channel的发送和接收主要考虑三种情况：

1. 移动缓冲区
2. 阻塞
3. 缓冲区复制 && 直接发送

#### 第一种情况：移动缓冲区
当channel缓冲区 **未满** 的时候，底层环形队列缓存将会保存消息，此时发送方和接收方都从队列缓存中**直接**操作消息。

* 当channel新增发送方，`hchan`首先会**加锁**，其次将发送方的消息体存放进`sendx`，接着移动`sendx`指针，最后释放锁。
![channel新增发送方](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629152419.png)

* 当channel新增接收方，`hchan`首先会**加锁**，其次将`recvx`的消息体取出，接着移动`recvx`指针，最后释放锁。
![channel新增接收方](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629162413.png)

#### 第二种情况：阻塞
当channel缓冲区 **已满且仍有发送方** 或者 **已空且仍有接收方** 的时候，此时新增的发送方G/接收方G将进入**阻塞状态**。此时，阻塞的G将被包装进sudog结构体中，并以双向链表的形式保存下来。

* 当channel缓冲区已满且仍有发送方，新增的发送方G将进入阻塞状态，以sudog的形式加入`sendq`中。
![channel缓冲区已满，新增发送方](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629165913.png)

* 当channel缓冲区已空且仍有接收方，新增的接收方G将进入阻塞状态，以sudog的形式加入`recvq`中。
![channel缓冲区已空，新增接收方](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629200441.png)

#### 第三种情况：缓冲区复制 && 直接发送
在上述第二种情况的基础上，当channel **存在阻塞发送方且新增了接收方** 的时候，channel会将缓冲区的数据返回给接收方，同时将阻塞在双向链表队头的发送方唤醒，并将其消息复制进缓冲区。
![channel发送方阻塞，新增接收方](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629194539.png)

当channel **存在阻塞接收方且新增了发送方** 的时候，channel会唤醒阻塞的接收方，然后直接把数据从发送方复制到接收方，**无需再经过缓冲区**，即直接发送。
![channel接收方阻塞，新增发送方](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629200728.png)

### 源码分析
接下来，我们通过源码的方式进一步分析channel的发送和接收。
```go
package main

func main() {
	ch := make(chan struct{})

	go func() {
		ch <- struct{}{}
	}()

	<-ch
}
```
```bash
"".main STEXT size=133 args=0x0 locals=0x28 funcid=0x0
	0x0000 00000 (main.go:3)	TEXT	"".main(SB), ABIInternal, $40-0
	0x0000 00000 (main.go:3)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:3)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:3)	PCDATA	$0, $-2
	0x000d 00013 (main.go:3)	JLS	121
	0x000f 00015 (main.go:3)	PCDATA	$0, $-1
	0x000f 00015 (main.go:3)	SUBQ	$40, SP
	0x0013 00019 (main.go:3)	MOVQ	BP, 32(SP)
	0x0018 00024 (main.go:3)	LEAQ	32(SP), BP
	0x001d 00029 (main.go:3)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x001d 00029 (main.go:3)	FUNCDATA	$1, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x001d 00029 (main.go:4)	LEAQ	type.chan struct {}(SB), AX
	0x0024 00036 (main.go:4)	MOVQ	AX, (SP)
	0x0028 00040 (main.go:4)	MOVQ	$0, 8(SP)
	0x0031 00049 (main.go:4)	PCDATA	$1, $0
	0x0031 00049 (main.go:4)	CALL	runtime.makechan(SB)
	0x0036 00054 (main.go:4)	MOVQ	16(SP), AX
	0x003b 00059 (main.go:4)	MOVQ	AX, "".ch+24(SP)
	0x0040 00064 (main.go:6)	MOVL	$8, (SP)
	0x0047 00071 (main.go:6)	LEAQ	"".main.func1·f(SB), CX
	0x004e 00078 (main.go:6)	MOVQ	CX, 8(SP)
	0x0053 00083 (main.go:6)	PCDATA	$1, $1
	0x0053 00083 (main.go:6)	CALL	runtime.newproc(SB)
	0x0058 00088 (main.go:10)	MOVQ	"".ch+24(SP), AX
	0x005d 00093 (main.go:10)	MOVQ	AX, (SP)
	0x0061 00097 (main.go:10)	MOVQ	$0, 8(SP)
	0x006a 00106 (main.go:10)	PCDATA	$1, $0
	0x006a 00106 (main.go:10)	CALL	runtime.chanrecv1(SB) # channel接收
	0x006f 00111 (main.go:11)	MOVQ	32(SP), BP
	0x0074 00116 (main.go:11)	ADDQ	$40, SP
	0x0078 00120 (main.go:11)	RET
	0x0079 00121 (main.go:11)	NOP
	0x0079 00121 (main.go:3)	PCDATA	$1, $-1
	0x0079 00121 (main.go:3)	PCDATA	$0, $-2
	0x0079 00121 (main.go:3)	CALL	runtime.morestack_noctxt(SB)
	0x007e 00126 (main.go:3)	PCDATA	$0, $-1
	0x007e 00126 (main.go:3)	NOP
	0x0080 00128 (main.go:3)	JMP	0
  ... 省略不相关的部分

"".main.func1 STEXT size=71 args=0x8 locals=0x18 funcid=0x0
	0x0000 00000 (main.go:6)	TEXT	"".main.func1(SB), ABIInternal, $24-8
	0x0000 00000 (main.go:6)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:6)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:6)	PCDATA	$0, $-2
	0x000d 00013 (main.go:6)	JLS	64
	0x000f 00015 (main.go:6)	PCDATA	$0, $-1
	0x000f 00015 (main.go:6)	SUBQ	$24, SP
	0x0013 00019 (main.go:6)	MOVQ	BP, 16(SP)
	0x0018 00024 (main.go:6)	LEAQ	16(SP), BP
	0x001d 00029 (main.go:6)	FUNCDATA	$0, gclocals·1a65e721a2ccc325b382662e7ffee780(SB)
	0x001d 00029 (main.go:6)	FUNCDATA	$1, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x001d 00029 (main.go:7)	MOVQ	"".ch+32(SP), AX
	0x0022 00034 (main.go:7)	MOVQ	AX, (SP)
	0x0026 00038 (main.go:7)	LEAQ	""..autotmp_1+16(SP), AX
	0x002b 00043 (main.go:7)	MOVQ	AX, 8(SP)
	0x0030 00048 (main.go:7)	PCDATA	$1, $1
	0x0030 00048 (main.go:7)	CALL	runtime.chansend1(SB) # channel发送
	0x0035 00053 (main.go:8)	MOVQ	16(SP), BP
	0x003a 00058 (main.go:8)	ADDQ	$24, SP
	0x003e 00062 (main.go:8)	RET
	0x003f 00063 (main.go:8)	NOP
	0x003f 00063 (main.go:6)	PCDATA	$1, $-1
	0x003f 00063 (main.go:6)	PCDATA	$0, $-2
	0x003f 00063 (main.go:6)	NOP
	0x0040 00064 (main.go:6)	CALL	runtime.morestack_noctxt(SB)
	0x0045 00069 (main.go:6)	PCDATA	$0, $-1
	0x0045 00069 (main.go:6)	JMP	0
  ... 省略不相关的部分
```

从汇编结果来看，channel的发送底层实现是`runtime.chansend1`函数，channel的接收底层实现是`runtime.chanrecv1`函数。我们进一步查看相关源码：
```go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}
```
值得注意的是，runtime源码中包括**2个recv**函数：`runtime.chanrecv1`和`runtime.chanrecv2`。从实现上面可以看出，`runtime.chanrecv2`是用于接收方返回两个参数的场景。

#### 发送
我们先来解析`runtime.chansend`的源码部分（PS：由于源码分析比较复杂，因此主要阐述其中的关键环节）：
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  ... 省略不相关的部分

  // 直接发送：当存在阻塞的接收方，且新增了发送方，那么直接把发送方的消息复制给接收方
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

  // 缓冲区：当缓冲区未满，那么通过移动环形队列缓存的指针来存储消息
  if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++ // 移动指针
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

  if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
  // 阻塞：当缓冲区已满，此时当前的发送方需要进入阻塞状态。
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
 	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2) // 将goroutine阻塞
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

  ... 省略不相关的部分
}
```

#### 接收
最后我们来解析`runtime.chanrecv`源码部分：
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  ... 省略不相关的部分

  // 缓冲区复制：当存在阻塞的发送方，会从缓冲区读取消息，并且唤醒发送方将消息放入缓冲区。这里面同时存在一个临界状态，即当缓冲区长度是0的时候，则会将发送方的消息直接复制到接收方
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

  // 缓冲区：当缓冲区未空的时候，那么通过移动环形队列缓存的指针来发送消息
  if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++ // 移动指针
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

  if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
  // 阻塞：当缓冲区已空，那么需要阻塞当前的接收方
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2) // 将goroutine阻塞

  ... 省略不相关的部分
}
```
对照一下`runtime.recv`源码实现：
```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 { // 缓冲区为0的情况，直接复制
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
    // 缓冲区不为0的情况，需要从缓冲区去发送消息，并且把阻塞队列队头的发送方唤醒
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

## channel的关闭
channel的关闭并不复杂。但由于channel本身是悲观锁实现，因此channel在关闭的时候，为了减少悲观锁的占用时间，channel会创建一个`glist`队列，将当前仍然阻塞的`sendq`和`recvq`上面的sudog加入`glist`，然后快速释放掉锁。紧接着，channel会将`glist`上面阻塞的goroutine以此唤醒。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-channel/20210629222355.png)

我们进一步分析channel的源码实现：
```go
package main

func main() {
	ch := make(chan struct{})

	go func() {
		ch <- struct{}{}

		close(ch)
	}()

	<-ch
}
```
```bash
$ go tool compile -S main.go

"".main.func1 STEXT size=86 args=0x8 locals=0x18 funcid=0x0
	0x0000 00000 (main.go:6)	TEXT	"".main.func1(SB), ABIInternal, $24-8
	0x0000 00000 (main.go:6)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:6)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:6)	PCDATA	$0, $-2
	0x000d 00013 (main.go:6)	JLS	79
	0x000f 00015 (main.go:6)	PCDATA	$0, $-1
	0x000f 00015 (main.go:6)	SUBQ	$24, SP
	0x0013 00019 (main.go:6)	MOVQ	BP, 16(SP)
	0x0018 00024 (main.go:6)	LEAQ	16(SP), BP
	0x001d 00029 (main.go:6)	FUNCDATA	$0, gclocals·1a65e721a2ccc325b382662e7ffee780(SB)
	0x001d 00029 (main.go:6)	FUNCDATA	$1, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x001d 00029 (main.go:7)	MOVQ	"".ch+32(SP), AX
	0x0022 00034 (main.go:7)	MOVQ	AX, (SP)
	0x0026 00038 (main.go:7)	LEAQ	""..autotmp_1+16(SP), CX
	0x002b 00043 (main.go:7)	MOVQ	CX, 8(SP)
	0x0030 00048 (main.go:7)	PCDATA	$1, $0
	0x0030 00048 (main.go:7)	CALL	runtime.chansend1(SB)
	0x0035 00053 (main.go:9)	MOVQ	"".ch+32(SP), AX
	0x003a 00058 (main.go:9)	MOVQ	AX, (SP)
	0x003e 00062 (main.go:9)	PCDATA	$1, $1
	0x003e 00062 (main.go:9)	NOP
	0x0040 00064 (main.go:9)	CALL	runtime.closechan(SB) # 关闭channel
	0x0045 00069 (main.go:10)	MOVQ	16(SP), BP
	0x004a 00074 (main.go:10)	ADDQ	$24, SP
	0x004e 00078 (main.go:10)	RET
	0x004f 00079 (main.go:10)	NOP
	0x004f 00079 (main.go:6)	PCDATA	$1, $-1
	0x004f 00079 (main.go:6)	PCDATA	$0, $-2
	0x004f 00079 (main.go:6)	CALL	runtime.morestack_noctxt(SB)
	0x0054 00084 (main.go:6)	PCDATA	$0, $-1
	0x0054 00084 (main.go:6)	JMP	0
  ... 省略不相关的部分
```
从汇编的结果可以看出，channel的关闭底层实现是`runtime.closechan`，让我们进一步查看相关源码：
```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList // 新建glist，是一个队列结构

	// release all readers
  // 把所有仍然阻塞的接收方加入glist
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
  // 把所有仍然阻塞的发送方加入glist
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
  // 将阻塞状态的goroutine唤醒
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3) // 唤醒goroutine
	}
}
```

channel的关闭遵循**两大原则**：

1. **channel并不一定要关闭**。Go语言本身可以通过GC来回收channel的内存空间，因此channel不关闭的情况也经常存在。但当不关闭channel引发goroutine无法正常退出的时候，我们需要考虑尽可能地关闭channel以解决**goroutine泄露问题**。
2. 不要在接收方关闭channel，而**要在发送方关闭channel，并发存在多个发送方的时候除外**。

## 无锁channel
**锁**是一种常见的并发控制技术，我们通常将锁的类型分为**乐观锁**和**悲观锁**。而所谓的无锁channel更准确的说是**基于乐观锁的实现**，也即基于[CAS（Compare And Swap）](https://en.wikipedia.org/wiki/Compare-and-swap)实现。

Go社区曾在2014年提出无锁channel的提案[^1]，但是该提案虽然很早就提出，但至今仍然没有被接受。主要有如下原因：

1. Go官方希望channel能够符合FIFO的顺序被唤醒，保证数据公平性，这是未使用无锁channel的最主要原因。
2. 无锁channel并不是无等待算法，是否能够有效提高channel在大规模应用的性能并没有得到验证。一个社区的实现[^2]甚至比基于futex的channel还要慢。
3. 无锁channel的可维护性非常差。

## 参考
1. [Go语言设计与实现：Channel](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)
2. [Go夜读：channel & select源码分析](https://talkgo.org/t/topic/75)
3. [Go语言原本](https://golang.design/under-the-hood/zh-cn/part2runtime/ch09lang/chan/#channel-4)

## 总结
本文通过图示+源码的方式深度解析了Go语言核心数据结构channel的工作原理，Go的并发设计基于CSP理论，而channel正是连通goroutine的桥梁。

## 脚注
[^1]: https://docs.google.com/document/d/1yIAYmbvL3JxOKOjuCyon7JhW4cSv1wy5hC0ApeGMV9s/pub
[^2]: https://github.com/OneOfOne/lfchan/blob/master/lfchan.go
