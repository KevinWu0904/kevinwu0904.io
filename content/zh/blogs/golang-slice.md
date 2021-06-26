---
title: Go源码学习之slice
toc: true
authors:
  - kevinwu
tags:
  - golang
series:
  - 深入理解Go语言
date: '2021-06-02'
lastmod: '2021-06-02'
draft: false
---

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-common/golang-logo.png)

slice是golang开发中最常用到的内置类型之一。与数组相比，它具有**长度不固定、可动态添加元素**的特性。

## 版本说明
本文涉及到的源码解析部分来源于go官方的[**1.15.8**](https://github.com/golang/go/tree/go1.15.8)版本。

## 定义
slice在golang的runtime层面定义如下：
```go
type slice struct {
	array unsafe.Pointer // 数组元素
	len   int // 长度
	cap   int // 容量
}
```
slice包括3个字段：

1. array：表示当前slice存放的数组元素。
2. len：表示当前slice已经使用的长度。
3. cap：表示当前slice的总容量，即最大能够容纳的元素个数。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-slice/20210601161955.png)

如图所示，slice的动态特性正是基于这样的数据结构：一方面，记录已经使用的元素个数；另一方面，元素存放在有容量上限的底层数组中。

## 创建slice
slice的创建需要借助golang builtin包中的`make`函数：
```go
// The make built-in function allocates and initializes an object of type
// slice, map, or chan (only). Like new, the first argument is a type, not a
// value. Unlike new, make's return type is the same as the type of its
// argument, not a pointer to it. The specification of the result depends on
// the type:
//	Slice: The size specifies the length. The capacity of the slice is
//	equal to its length. A second integer argument may be provided to
//	specify a different capacity; it must be no smaller than the
//	length. For example, make([]int, 0, 10) allocates an underlying array
//	of size 10 and returns a slice of length 0 and capacity 10 that is
//	backed by this underlying array.
//	Map: An empty map is allocated with enough space to hold the
//	specified number of elements. The size may be omitted, in which case
//	a small starting size is allocated.
//	Channel: The channel's buffer is initialized with the specified
//	buffer capacity. If zero, or the size is omitted, the channel is
//	unbuffered.
func make(t Type, size ...IntegerType) Type
```

`make`在golang中是一个创建内置类型的通用函数，不仅仅是slice，包括chan、map都需要借助`make`函数来创建。slice的创建主要包括如下两种形式：
```go
nums := make([]int, 10) // 创建长度和容量均为10的int slice
nums := make([]int, 0, 10) // 创建长度为0，容量为10的int slice
```

以下面的代码为例：
```go
package main

import (
    "fmt"
)

func main() {
    nums := make([]int, 0, 10)
    nums = append(nums, 0)
    fmt.Println(nums)
}

```

让我们通过go的官方工具来查看汇编结果：
```bash
$ go tool compile -S main.go

"".main STEXT size=217 args=0x0 locals=0x58 funcid=0x0
	0x0000 00000 (main.go:7)	TEXT	"".main(SB), ABIInternal, $88-0
	0x0000 00000 (main.go:7)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:7)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:7)	PCDATA	$0, $-2
	0x000d 00013 (main.go:7)	JLS	207
	0x0013 00019 (main.go:7)	PCDATA	$0, $-1
	0x0013 00019 (main.go:7)	SUBQ	$88, SP
	0x0017 00023 (main.go:7)	MOVQ	BP, 80(SP)
	0x001c 00028 (main.go:7)	LEAQ	80(SP), BP
	0x0021 00033 (main.go:7)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$1, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$2, "".main.stkobj(SB)
	0x0021 00033 (main.go:8)	LEAQ	type.int(SB), AX
	0x0028 00040 (main.go:8)	MOVQ	AX, (SP)
	0x002c 00044 (main.go:8)	MOVQ	$0, 8(SP)
	0x0035 00053 (main.go:8)	MOVQ	$10, 16(SP)
	0x003e 00062 (main.go:8)	PCDATA	$1, $0
	0x003e 00062 (main.go:8)	NOP
	0x0040 00064 (main.go:8)	CALL	runtime.makeslice(SB) # 从这一行我们可以看出slice的创建最终底层实现是runtime.makeslice函数
	0x0045 00069 (main.go:8)	MOVQ	24(SP), AX
	0x004a 00074 (main.go:9)	MOVQ	$0, (AX)
	0x0051 00081 (main.go:10)	MOVQ	AX, (SP)
	0x0055 00085 (main.go:10)	MOVQ	$1, 8(SP)
	0x005e 00094 (main.go:10)	MOVQ	$10, 16(SP)
	0x0067 00103 (main.go:10)	CALL	runtime.convTslice(SB)
	0x006c 00108 (main.go:10)	MOVQ	24(SP), AX
	0x0071 00113 (main.go:10)	XORPS	X0, X0
	0x0074 00116 (main.go:10)	MOVUPS	X0, ""..autotmp_13+64(SP)
	0x0079 00121 (main.go:10)	LEAQ	type.[]int(SB), CX
	0x0080 00128 (main.go:10)	MOVQ	CX, ""..autotmp_13+64(SP)
	0x0085 00133 (main.go:10)	MOVQ	AX, ""..autotmp_13+72(SP)
	... 省略不相关部分
```
PS：请读者先不要迷失在汇编代码的细节中，目前我们只需要关注"(main.go:8)"对应底层实现是调用go的`runtime.makeslice`函数，了解这一点即可。

因此，我们进一步去翻阅相关的go源码：
```go
func makeslice(et *_type, len, cap int) unsafe.Pointer { // _type是go所有类型在runtime层面的表示形式
	mem, overflow := math.MulUintptr(et.size, uintptr(cap)) // 预期内存开销是：元素类型长度x元素个数，同时判断是否算术溢出
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true) // 申请内存块
}
```
`makeslice`函数计算出所需的内存开销之后，会检查是否算术溢出，紧接着通过`mallocgc`函数去申请内存块。PS：由于mallocgc涉及到比较多的go内存管理相关的知识点，因此不在本文展开。

至此，slice就完成了内存申请。

## 复制slice
slice的复制可以借助golang builtin包中的`copy`函数：
```go
// The copy built-in function copies elements from a source slice into a
// destination slice. (As a special case, it also will copy bytes from a
// string to a slice of bytes.) The source and destination may overlap. Copy
// returns the number of elements copied, which will be the minimum of
// len(src) and len(dst).
func copy(dst, src []Type) int
```

以下面的代码为例：
```go
package main

import (
    "fmt"
)

func main() {
    src := []int{ 1, 2, 3 }
    tar := make([]int, 2)
    copy(tar, src)
    fmt.Println(tar)
}

```

让我们通过go的官方工具来查看汇编结果：
```bash
$ go tool compile -S main.go

"".main STEXT size=261 args=0x0 locals=0x70 funcid=0x0
	0x0000 00000 (main.go:7)	TEXT	"".main(SB), ABIInternal, $112-0
	0x0000 00000 (main.go:7)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:7)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:7)	PCDATA	$0, $-2
	0x000d 00013 (main.go:7)	JLS	249
	0x0013 00019 (main.go:7)	PCDATA	$0, $-1
	0x0013 00019 (main.go:7)	SUBQ	$112, SP
	0x0017 00023 (main.go:7)	MOVQ	BP, 104(SP)
	0x001c 00028 (main.go:7)	LEAQ	104(SP), BP
	0x0021 00033 (main.go:7)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$1, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$2, "".main.stkobj(SB)
	0x0021 00033 (main.go:8)	MOVQ	$0, ""..autotmp_12+64(SP)
	0x002a 00042 (main.go:8)	XORPS	X0, X0
	0x002d 00045 (main.go:8)	MOVUPS	X0, ""..autotmp_12+72(SP)
	0x0032 00050 (main.go:8)	MOVQ	$1, ""..autotmp_12+64(SP)
	0x003b 00059 (main.go:8)	MOVQ	$2, ""..autotmp_12+72(SP)
	0x0044 00068 (main.go:8)	MOVQ	$3, ""..autotmp_12+80(SP)
	0x004d 00077 (main.go:9)	LEAQ	type.int(SB), AX
	0x0054 00084 (main.go:9)	MOVQ	AX, (SP)
	0x0058 00088 (main.go:9)	MOVQ	$2, 8(SP)
	0x0061 00097 (main.go:9)	MOVQ	$3, 16(SP)
	0x006a 00106 (main.go:9)	LEAQ	""..autotmp_12+64(SP), AX
	0x006f 00111 (main.go:9)	MOVQ	AX, 24(SP)
	0x0074 00116 (main.go:9)	PCDATA	$1, $0
	0x0074 00116 (main.go:9)	CALL	runtime.makeslicecopy(SB) # 调用runtime的makeslicecopy函数
	0x0079 00121 (main.go:9)	MOVQ	32(SP), AX
	0x007e 00126 (main.go:11)	MOVQ	AX, (SP)
	0x0082 00130 (main.go:11)	MOVQ	$2, 8(SP)
	0x008b 00139 (main.go:11)	MOVQ	$2, 16(SP)
	0x0094 00148 (main.go:11)	CALL	runtime.convTslice(SB)
	0x0099 00153 (main.go:11)	MOVQ	24(SP), AX
	0x009e 00158 (main.go:11)	XORPS	X0, X0
	0x00a1 00161 (main.go:11)	MOVUPS	X0, ""..autotmp_17+88(SP)
	0x00a6 00166 (main.go:11)	LEAQ	type.[]int(SB), CX
	0x00ad 00173 (main.go:11)	MOVQ	CX, ""..autotmp_17+88(SP)
	0x00b2 00178 (main.go:11)	MOVQ	AX, ""..autotmp_17+96(SP)
	... 省略不相关部分
```

通过汇编代码的结果可以看出，`copy`函数在go的runtime层面是调用了`runtime.makeslicecopy`函数。因此，我们进一步查看相关源码：
```go
// makeslicecopy allocates a slice of "tolen" elements of type "et",
// then copies "fromlen" elements of type "et" into that new allocation from "from".
func makeslicecopy(et *_type, tolen int, fromlen int, from unsafe.Pointer) unsafe.Pointer { // 参数分别是：数据类型、目标长度、来源长度、来源元素数组
	/*
		计算需要复制的内存块大小：
		1. 目标slice长度 > 来源slice长度，以来源长度为准
		2. 目标slice长度 <= 来源slice长度，以目标长度为准
	*/
	var tomem, copymem uintptr
	if uintptr(tolen) > uintptr(fromlen) {
		var overflow bool
		tomem, overflow = math.MulUintptr(et.size, uintptr(tolen))
		if overflow || tomem > maxAlloc || tolen < 0 {
			panicmakeslicelen()
		}
		copymem = et.size * uintptr(fromlen)
	} else {
		// fromlen is a known good length providing and equal or greater than tolen,
		// thereby making tolen a good slice length too as from and to slices have the
		// same element width.
		tomem = et.size * uintptr(tolen)
		copymem = tomem
	}

	/*
		申请内存块
	*/ 
	var to unsafe.Pointer
	if et.ptrdata == 0 {
		to = mallocgc(tomem, nil, false)
		if copymem < tomem {
			memclrNoHeapPointers(add(to, copymem), tomem-copymem)
		}
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		to = mallocgc(tomem, et, true)
		if copymem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice to
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(to), uintptr(from), copymem)
		}
	}

	/*
		数据竞争检测和支持与内存清理程序的互操作
	*/
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(makeslicecopy)
		racereadrangepc(from, copymem, callerpc, pc)
	}
	if msanenabled {
		msanread(from, copymem)
	}

	// 复制内存块
	memmove(to, from, copymem)

	return to
}
```

小结一下，通过`copy`函数的底层实现，我们能够知道：

1. slice的复制是基于复制内存块来完成的。
2. slice的复制会考虑目标slice的长度，仅会复制不大于目标slice长度的元素。

## 新增元素
给slice新增元素需要借助golang builtin包中的`append`函数：
```go
// The append built-in function appends elements to the end of a slice. If
// it has sufficient capacity, the destination is resliced to accommodate the
// new elements. If it does not, a new underlying array will be allocated.
// Append returns the updated slice. It is therefore necessary to store the
// result of append, often in the variable holding the slice itself:
//	slice = append(slice, elem1, elem2)
//	slice = append(slice, anotherSlice...)
// As a special case, it is legal to append a string to a byte slice, like this:
//	slice = append([]byte("hello "), "world"...)
func append(slice []Type, elems ...Type) []Type
```

日常使用示例如下：
```go
nums := make([]int, 0, 10)
nums = append(nums, 1)
nums = append(nums, 5)
nums = append(nums, 6)
```

## 删除元素
不同于新增元素，golang中并未内置删除slice指定元素的函数，需要开发者自己去实现。就笔者所了解的删除元素方法主要有如下几种方式：

方法一：创建新的slice切片
```go
func removeElem(src []int, toDel int) []int {
	tar := make([]int, 0, len(src))
	for _, num := range src {
		if num != toDel {
			tar = append(tar, num)
		}
	}

	return tar
}
```

方法二：复用原有的slice切片
```go
func removeElem(src []int, toDel int) []int {
	tar := src[:0]
	for _, num := range src {
		if num != toDel {
			tar = append(tar, num)
		}
	}

	return tar
}
```

方法三：复用原有的slice切片，平移剩下的元素
```go
func removeElem(src []int, toDel int) []int {
	for i := 0; i < len(src); i++ {
		if src[i] == toDel {
			src = append(src[:i], src[i+1:]...)
			i--
		}
	}
	
	return src
}
```

小结一下，方法一的主要优点是不修改原有slice的数据，仅当不允许修改源slice数据的前提下推荐方法一。其它情况下，请使用方法二或者方法三，性能方面和开销方面都比较优秀。

## 扩容slice
动态扩容是slice最大的特点。我们日常使用slice的过程中，并不会感受到slice的真实容量变化情况，golang底层会帮助我们完成对应的slice扩容。

以下面的代码为例：
```go
package main

import (
    "fmt"
)

func main() {
    nums := make([]int, 0, 2) // 初始设置为2
    nums = append(nums, 1)
    nums = append(nums, 2)
    nums = append(nums, 3) // 触发扩容
    nums = append(nums, 4)
    nums = append(nums, 5)
    fmt.Println(nums)
}

```

让我们通过go的官方工具来查看汇编结果：
```bash
$ go tool compile -S main.go

"".main STEXT size=490 args=0x0 locals=0x60 funcid=0x0
	0x0000 00000 (main.go:7)	TEXT	"".main(SB), ABIInternal, $96-0
	0x0000 00000 (main.go:7)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:7)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:7)	PCDATA	$0, $-2
	0x000d 00013 (main.go:7)	JLS	480
	0x0013 00019 (main.go:7)	PCDATA	$0, $-1
	0x0013 00019 (main.go:7)	SUBQ	$96, SP
	0x0017 00023 (main.go:7)	MOVQ	BP, 88(SP)
	0x001c 00028 (main.go:7)	LEAQ	88(SP), BP
	0x0021 00033 (main.go:7)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$1, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$2, "".main.stkobj(SB)
	0x0021 00033 (main.go:8)	LEAQ	type.int(SB), AX
	0x0028 00040 (main.go:8)	MOVQ	AX, (SP)
	0x002c 00044 (main.go:8)	MOVQ	$0, 8(SP)
	0x0035 00053 (main.go:8)	MOVQ	$2, 16(SP)
	0x003e 00062 (main.go:8)	PCDATA	$1, $0
	0x003e 00062 (main.go:8)	NOP
	0x0040 00064 (main.go:8)	CALL	runtime.makeslice(SB)
	0x0045 00069 (main.go:8)	MOVQ	24(SP), AX
	0x004a 00074 (main.go:9)	MOVQ	$1, (AX)
	0x0051 00081 (main.go:10)	MOVQ	$2, 8(AX)
	0x0059 00089 (main.go:11)	LEAQ	type.int(SB), CX
	0x0060 00096 (main.go:11)	MOVQ	CX, (SP)
	0x0064 00100 (main.go:11)	MOVQ	AX, 8(SP)
	0x0069 00105 (main.go:11)	MOVQ	$2, 16(SP)
	0x0072 00114 (main.go:11)	MOVQ	$2, 24(SP)
	0x007b 00123 (main.go:11)	MOVQ	$3, 32(SP)
	0x0084 00132 (main.go:11)	CALL	runtime.growslice(SB) # 调用runtime的growslice函数
	0x0089 00137 (main.go:11)	MOVQ	40(SP), AX
	0x008e 00142 (main.go:11)	MOVQ	56(SP), CX
	0x0093 00147 (main.go:11)	MOVQ	48(SP), DX
	0x0098 00152 (main.go:11)	MOVQ	$3, 16(AX)
	0x00a0 00160 (main.go:11)	INCQ	DX
	0x00a3 00163 (main.go:12)	LEAQ	1(DX), BX
	0x00a7 00167 (main.go:12)	CMPQ	CX, BX
	0x00aa 00170 (main.go:12)	JCS	403
	0x00b0 00176 (main.go:12)	MOVQ	$4, (AX)(DX*8)
	0x00b8 00184 (main.go:13)	LEAQ	1(BX), DX
	0x00bc 00188 (main.go:13)	NOP
	0x00c0 00192 (main.go:13)	CMPQ	CX, DX
	0x00c3 00195 (main.go:13)	JCS	325
	0x00c9 00201 (main.go:13)	MOVQ	$5, (AX)(BX*8)
	0x00d1 00209 (main.go:14)	MOVQ	AX, (SP)
	0x00d5 00213 (main.go:14)	MOVQ	DX, 8(SP)
	0x00da 00218 (main.go:14)	MOVQ	CX, 16(SP)
	0x00df 00223 (main.go:14)	NOP
	0x00e0 00224 (main.go:14)	CALL	runtime.convTslice(SB)
	0x00e5 00229 (main.go:14)	MOVQ	24(SP), AX
	... 省略不相关部分
```

因此，我们进一步查看相关源码：
```go
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	/*
		扩容规则：
		1. 长度 < 1024的情况下，扩容2倍
		2. 长度 > 1024的情况下，扩容1.25倍
	*/ 

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	/*
		内存对齐操作，对齐之后的容量相比于上述规则来说会有所偏差！
	*/
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	/*
		分配内存块
	*/
	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
		}
	}

	/*
		把旧slice的内容复制到新slice中
	*/
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

小结一下，通过`growslice`函数的底层实现，我们能够知道：

1. `append`操作在golang编译结果中会转化成`runtime.growslice`函数，对上层透明，因而开发者无需关心slice底层容量大小的问题。
2. slice触发扩容操作会创建新的容量数组，将旧的slice内容复制过去，开发者需要关注**有多个变量引用同一个slice且发生扩容**的情况。
3. slice的扩容原则是：对于长度 < 1024的情况下，2倍增长；对于长度 > 1024的情况下，1.25倍增长。但由于存在**内存对齐**，slice的容量在扩容结束后会有所偏差！

## 总结
本文详细介绍了Go的内置类型slice，以常见的代码为例，通过go官方工具的汇编结果解析了底层实现。另一方面，本文从源码角度得出一些开发者较为容易迷惑的语法问题，例如slice的复制长度问题、slice扩容之后的底层数组更替问题、slice扩容容量大小问题等等。

总的来说，源码学习是一条漫漫长路。笔者认为开发者既不应该无视源码而只靠背诵一些结论来学习语言，也不应该过分关注源码细节导致无从入手。本文从slice的源码中，也牵扯出一些未能深入解析的概念：Go的runtime、Plan 9汇编、内存对齐等，关于这些概念的进一步解析则会随着笔者的深入学习而逐步完善，在此也希望能与读者共勉！