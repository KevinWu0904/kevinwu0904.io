---
title: 谈一谈Golang Error Handling
toc: true
authors:
  - kevinwu
tags:
  - golang
date: '2021-04-12'
lastmod: '2021-04-12'
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