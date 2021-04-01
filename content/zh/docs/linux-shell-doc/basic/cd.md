---
title: cd
toc: true
authors:
  - kevinwu
date: '2021-03-31'
lastmod: '2021-03-31'
draft: false
---

## 功能概述
cd用于在Linux目录中跳转。

## 使用示例
* 示例一：进入gogogo目录
```
kevinwu@debian:~$ cd $GOPATH/src/gogogo
kevinwu@debian:~/go/src/gogogo$
```

* 示例二：返回父目录
```
kevinwu@debian:~/go/src/gogogo$ cd ..
kevinwu@debian:~/go/src$
```

* 示例三：返回到上一次所在目录
```
kevinwu@debian:~/go/src$ cd -
/home/kevinwu/go/src/gogogo # 同时也输出上一次目录名称
kevinwu@debian:~/go/src/gogogo$
```

* 示例四：返回主目录
```
kevinwu@debian:~/go/src/gogogo$ cd
kevinwu@debian:~$
```
或
```
kevinwu@debian:~/go/src/gogogo$ cd ~
kevinwu@debian:~$
```