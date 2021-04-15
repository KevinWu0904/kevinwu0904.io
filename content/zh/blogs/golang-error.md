---
title: 谈一谈Golang Error Handling
toc: true
authors:
  - kevinwu
tags:
  - golang
date: '2021-04-15'
lastmod: '2021-04-15'
draft: false
---

![](/images/blogs-golang-common/golang-logo.png)

Golang的Error Handling一直备受争议，官方和社区也持续提出各项改进方案。作为语言级别的error支持，Go对error的实现却异常简单，甚至可以说是简陋。那么，到底什么样的做法才是Golang Error Handling的最佳实践呢？

## Errors are values
“[Errors are values](https://blog.golang.org/errors-are-values)”源自于Go语言创始人之一的Rob Pike对error的设计理念。Rob Pike认为，error和函数的其它返回值地位相等，只是多个返回值的其中之一，并无任何特殊之处。因此，对error的处理就如同正常对待一个函数的返回值一样进行。

理解了Rob Pike的设计理念，就能够自然理解为什么在Go语言中会反复出现：
```go
if err != nil {
	return err
}
```
因为error就是一个普通的返回值，代码流程中也对它进行了简单处理。只不过，对于Go语言的绝大部分情况来说，error的处理方式都只需要判断非空返回。所以，Go语言常因冗长而重复的error处理被吐槽。

### 定义
翻看Go语言的源码，其中error的定义非常简单：
```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```
但凡实现了`Error() string`方法的类型均可作为error返回。

### 使用
error的使用即如上述所说，只需要当成普通返回值去处理即可。Go语言对于error的设计不像Java对于Exception的设计，Go中没有try/catch机制。调用多个具有error返回值的函数，需要重复多次处理：
```go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
```

### 弊端
Go语言的error设计，在长期的实践中备受争议。开发者主要吐槽的点有：

1. error处理穿插在Go语言的代码中，割裂正常的逻辑代码，可读性受到影响。
2. 大量重复的`if err != nil`片段却无法化简。
3. 简单的`return err`不能适用所有场景。

## Error Handling
本文的Error Handling主要是针对上述弊端中的第3点而言，旨在探索出一套简洁、通用、有效的error处理最佳实践。而针对上述弊端中的第1点和第2点，将在Go2 Draft Design中给出方案规划。

### Sentinel Error
在Go的标准库和许多三方Go框架中，我们都能够看到形如下面的ErrXXX全局变量：
```go
var ErrShortWrite = errors.New("short write")

var ErrShortBuffer = errors.New("short buffer")

var EOF = errors.New("EOF")

var ErrUnexpectedEOF = errors.New("unexpected EOF")

var ErrNoProgress = errors.New("multiple Read calls return no data or error")
```

使用时将全局变量作为返回值即可：
```go
if f.offset >= len(f.contents) {
  return 0, io.EOF
}
```

这里需要重点强调：**errors.New()方法即使创建相同内容的error，也不是同一个error**。那么这是什么原理？

让我们来看一下errors.New的源码实现：
```go
package errors

// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text} // 注意这里：返回的是&errorString{text}指针，而不是errorString{text}值
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

注意这里：返回的是&errorString{text}指针，而不是errorString{text}值。在Go语言中，指针的等值比较依据是地址。因此，即使两个error.New内容相同，等值比较也只会返回false。

Sentinel Error推荐在通用的标准库或第三方框架提供，适合用于Error产生的源头。

Sentinel Error因位于较为底层的位置，能够比较直观的反馈出error的根因，便于快速定位问题。但它的缺点也因此有关，滥用Sentinel Error会导致应用上层接收到底层过于抽象的error，从而产生信息迷惑。（例如：遇到“file does not exist”，虽然能大致归因，但不利于判断具体是哪个代码段产生的）

这个时候，我们往往不能够再简单的`return err`，需要附加一些信息。

### go1.13前的fmt.Errorf
在Go1.13版本前，通过`fmt.Errorf`可以简单包装一个error类型，返回新的error类型。

具体用法如下：
```go
if err == os.ErrNotExist {
	return fmt.Errorf("xxx.go meet err: %v", err)
}
```

通过`fmt.Errorf`包装，原有的error类型将会丢失，因此无法继续使用`==`来比较。那么如果希望保留error的原始类型，用于判断和处理，应该如何完成呢？

### go1.13的Error Wrapping

* Wrap

Go1.13版本提出了[“Error Wrapping”](https://golang.org/doc/go1.13#error_wrapping)的概念，通过增强`fmt.Errorf`的能力来保留error的原始类型。具体的使用示例如下：
```go
func top() error {
	if err := middle(); err != nil {
		return fmt.Errorf("error wrapper 2 : %w", err)
	}

	return nil
}

func middle() error {
	if err := bottom(); err != nil {
		return fmt.Errorf("error wrapper 1 : %w", err)
	}

	return nil
}

func bottom() error {
	return errors.New("core error")
}
```
不注意看的话，会以为和之前版本的`fmt.Errorf`一样。这里的区别是在`%v`和`%w`这两个不同的占位符上面，`%w`是Go1.13版本新增的占位符类型，内部通过结构体嵌套来记录error的原始类型。

![](/images/blogs-golang-error/wrap.png)

* Unwrap

与Wrap相对应的是Unwrap。Go1.13版本的标准库errors中提供了`Unwrap`方法，每调用一次`Unwrap`都能够剥离一层error类型。
```go
err := top()

for err != nil {
	t.Log(err)
	err = errors.Unwrap(err)
}
	
// Output
// error wrapper 2 : error wrapper 1 : core error
// error wrapper 1 : core error
// core error
```

![](/images/blogs-golang-error/unwrap.png)

* Is和As

`Is`和`As`是Go1.13版本errors包中提供的另外2个核心方法：`Is`用于判断src error和target error是否同属一个Error Wrapping，`As`用于将同属一个Error Wrapping的src error转换为target error。

`Is`使用示例：
```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func main() {
	if _, err := os.Open("non-existing"); err != nil {
		if errors.Is(err, os.ErrNotExist) {
			fmt.Println("file does not exist")
		} else {
			fmt.Println(err)
		}
	}
}
```

`As`使用示例：
```go
package main

import (
	"errors"
	"fmt"
	"os"
)

func main() {
	if _, err := os.Open("non-existing"); err != nil {
		var pathError *os.PathError
		if errors.As(err, &pathError) {
			fmt.Println("Failed at path:", pathError.Path)
		} else {
			fmt.Println(err)
		}
	}
}
```

### pkg/errors
Go1.13版本的Error Wrapping实际上是借鉴了社区开源库：[https://github.com/pkg/errors](https://github.com/pkg/errors)。不仅如此，pkg/errors还能够提供打印Error Stack的能力。

1. error源头使用errors.New
```go
func bottom() error {
	return errors.New("bottom error")
}
```

2. error调用链使用errors.Wrap
```go
func top() error {
	if err := middle(); err != nil {
		return errors.Wrap(err, "top error")
	}

	return nil
}

func middle() error {
	if err := bottom(); err != nil {
		return errors.Wrap(err, "middle error")
	}

	return nil
}
```

3. 获取error的RootCause和Error Stack
```go
func main() {
	fmt.Printf("%+v", errors.Cause(top()))
}

bottom error
main.bottom
        /Users/wuguohua/Workspace/Go/src/github.com/KevinWu0904/interview/main.go:30
main.middle
        /Users/wuguohua/Workspace/Go/src/github.com/KevinWu0904/interview/main.go:22
main.top
        /Users/wuguohua/Workspace/Go/src/github.com/KevinWu0904/interview/main.go:14
main.main
        /Users/wuguohua/Workspace/Go/src/github.com/KevinWu0904/interview/main.go:10
runtime.main
        /Users/wuguohua/.gvm/gos/go1.15.8/src/runtime/proc.go:204
runtime.goexit
        /Users/wuguohua/.gvm/gos/go1.15.8/src/runtime/asm_amd64.s:1374
Process finished with the exit code 0
```

4. 整体代码
```go
package main

import (
	"fmt"

	"github.com/pkg/errors"
)

func main() {
	fmt.Printf("%+v", errors.Cause(top()))
}

func top() error {
	if err := middle(); err != nil {
		return errors.Wrap(err, "top error")
	}

	return nil
}

func middle() error {
	if err := bottom(); err != nil {
		return errors.Wrap(err, "middle error")
	}

	return nil
}

func bottom() error {
	return errors.New("bottom error")
}
```

### 小结
Sentinel Error应该和Go1.13 Error Wrapping组合使用，而pkg/errors则推荐独立使用（因为pkg/errors内部也有errors.New方法）。

另一方面，pkg/errors可能会随着Go官方的error方案迭代进入deprecated状态。因此如果你的项目需要Error Stack信息，则推荐pkg/errors，否则目前来看推荐Go1.13的Error Wrapping。

## Go2 Draft Design
由于Go2 Error Handling仍然处于Proposal阶段[^1]，因此只对目前官方展示出的Draft Design简单解读。官方示例如下：

改造前：
```go
func printSum(a, b string) error {
	x, err := strconv.Atoi(a)
	if err != nil {
		return err
	}
	y, err := strconv.Atoi(b)
	if err != nil {
		return err
	}
	fmt.Println("result:", x + y)
	return nil
}
```

改造后：
```go
func printSum(a, b string) error {
	handle err { return err }
	x := check strconv.Atoi(a)
	y := check strconv.Atoi(b)
	fmt.Println("result:", x + y)
	return nil
}
```

Go2计划通过引入两个关键字`handle`和`check`来帮助化简error对代码逻辑的割裂和大量`if err != nil`的重复。

可以看到，改造后的代码分支显著减少，整体可读性大大提升。并且handle代码块对最终的error进行统一处理，减少重复逻辑。

不过，由于Go2仍未发布，因此目前的方案只能说是一个可能性的趋势和方向。期待未来的Go2能够成功化简error，那将会是广大Go开发者的福音！

## 总结
本文详细介绍了Go语言error的历史演进和当前的最佳实践，另外还简要提及Go2的error draft design。error的设计可以说是Go语言的一大特点，也是一大槽点，但随着时间的推移，笔者相信官方最终一定能够对这个问题提供最佳的解决方案。

## 脚注
[^1]: https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling.md