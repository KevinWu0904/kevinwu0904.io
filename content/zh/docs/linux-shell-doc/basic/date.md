---
title: date
toc: true
authors:
  - kevinwu
date: '2021-04-02'
lastmod: '2021-04-02'
draft: false
---

## 功能概述
date用于显示当前的系统时间。

## 使用示例
* 示例一：显示当前时间
```bash
kevinwu@debian:~$ date
2021年 04月 02日 星期五 22:15:05 CST
```

* 示例二：显示当前UTC时间
```bash
kevinwu@debian:~$ date --utc
2021年 04月 02日 星期五 14:18:18 UTC
```

* 示例三：显示当前时间，并以年月日数字形式展示
```bash
kevinwu@debian:~$ date +%Y%m%d
20210402
```