---
title: ls
toc: true
authors:
  - kevinwu
date: '2021-04-03'
lastmod: '2021-04-03'
draft: false
---

## 功能概述
ls用于展示当前目录的文件详情。

## 使用示例
* 示例一：显示文件详情
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls -l
总用量 100
-rw-r--r--  1 kevinwu kevinwu  1052 1月  22 12:51 appveyor.yml
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 assets
-rw-r--r--  1 kevinwu kevinwu 28773 1月  22 12:51 CHANGELOG.md
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 cmd
-rw-r--r--  1 kevinwu kevinwu  2440 1月  22 12:51 CONTRIBUTING.md
drwxr-xr-x  7 kevinwu kevinwu  4096 1月  22 12:51 Documentation
drwxr-xr-x 13 kevinwu kevinwu  4096 1月  22 12:51 _fixtures
-rw-r--r--  1 kevinwu kevinwu   863 1月  22 12:51 go.mod
-rw-r--r--  1 kevinwu kevinwu  7389 1月  22 12:51 go.sum
-rw-r--r--  1 kevinwu kevinwu   420 1月  22 12:51 ISSUE_TEMPLATE.md
-rw-r--r--  1 kevinwu kevinwu  1079 1月  22 12:51 LICENSE
-rw-r--r--  1 kevinwu kevinwu   565 1月  22 12:51 Makefile
drwxr-xr-x 12 kevinwu kevinwu  4096 1月  22 12:51 pkg
-rw-r--r--  1 kevinwu kevinwu  2152 1月  22 12:51 README.md
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 _scripts
drwxr-xr-x  9 kevinwu kevinwu  4096 1月  22 12:51 service
drwxr-xr-x  6 kevinwu kevinwu  4096 1月  22 12:51 vendor
```

界面说明：
1. 最左边1位字符表示**文件属性**。其中-表示普通文件，d表示目录，l表示软链接等等。
2. 紧接着3组rwx表示**文件权限**。其中r表示读权限，w表示写权限，x表示执行权限。3组不同的rwx分别对应着文件所属用户的权限，文件所属用户组的权限，其它用户的权限。
3. 紧接着1个数字表示**文件硬连接数或者目录子目录数**。对于普通文件记录硬连接数，对于目录来说记录子目录数（要包括.和..两个特殊目录）。
4. 接下来1个用户表示**文件所属用户**。
5. 接下来1个用户组表示**文件所属用户组**。
6. 接下来1个数字表示**文件大小**。单位是字节（Byte）。
7. 接下来1个时间表示**文件最新修改时间**。
8. 最后是1个字符串表示**文件名**。

* 示例二：显示非列表形式的文件概览
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls
appveyor.yml  CHANGELOG.md  CONTRIBUTING.md  _fixtures  go.sum             LICENSE   pkg        _scripts  vendor
assets        cmd           Documentation    go.mod     ISSUE_TEMPLATE.md  Makefile  README.md  service
```

* 示例三：显示包含隐藏文件的文件详情
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls -al
总用量 140
drwxr-xr-x 13 kevinwu kevinwu  4096 1月  22 12:51 .
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 ..
-rw-r--r--  1 kevinwu kevinwu  1052 1月  22 12:51 appveyor.yml
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 assets
-rw-r--r--  1 kevinwu kevinwu 28773 1月  22 12:51 CHANGELOG.md
-rw-r--r--  1 kevinwu kevinwu   173 1月  22 12:51 .cirrus.yml
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 cmd
-rw-r--r--  1 kevinwu kevinwu  2440 1月  22 12:51 CONTRIBUTING.md
-rw-r--r--  1 kevinwu kevinwu   239 1月  22 12:51 .deepsource.toml
drwxr-xr-x  7 kevinwu kevinwu  4096 1月  22 12:51 Documentation
drwxr-xr-x 13 kevinwu kevinwu  4096 1月  22 12:51 _fixtures
drwxr-xr-x  8 kevinwu kevinwu  4096 1月  22 12:51 .git
-rw-r--r--  1 kevinwu kevinwu    29 1月  22 12:51 .gitattributes
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 .github
-rw-r--r--  1 kevinwu kevinwu   105 1月  22 12:51 .gitignore
-rw-r--r--  1 kevinwu kevinwu   863 4月   3 21:07 go.mod
-rw-r--r--  1 kevinwu kevinwu  7389 1月  22 12:51 go.sum
-rw-r--r--  1 kevinwu kevinwu   420 1月  22 12:51 ISSUE_TEMPLATE.md
-rw-r--r--  1 kevinwu kevinwu  1079 1月  22 12:51 LICENSE
-rw-r--r--  1 kevinwu kevinwu   565 1月  22 12:51 Makefile
drwxr-xr-x 12 kevinwu kevinwu  4096 1月  22 12:51 pkg
-rw-r--r--  1 kevinwu kevinwu  2152 1月  22 12:51 README.md
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 _scripts
drwxr-xr-x  9 kevinwu kevinwu  4096 1月  22 12:51 service
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 .teamcity
-rw-r--r--  1 kevinwu kevinwu  1405 1月  22 12:51 .travis.yml
drwxr-xr-x  6 kevinwu kevinwu  4096 1月  22 12:51 vendor
```

* 示例四：递归显示列表形式的文件详情
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls -lR service/
service/:
总用量 56
drwxr-xr-x 2 kevinwu kevinwu 4096 1月  22 12:51 api
-rw-r--r-- 1 kevinwu kevinwu 8757 1月  22 12:51 client.go
-rw-r--r-- 1 kevinwu kevinwu 1219 1月  22 12:51 config.go
drwxr-xr-x 3 kevinwu kevinwu 4096 1月  22 12:51 dap
drwxr-xr-x 2 kevinwu kevinwu 4096 1月  22 12:51 debugger
-rw-r--r-- 1 kevinwu kevinwu 1707 1月  22 12:51 listenerpipe.go
drwxr-xr-x 2 kevinwu kevinwu 4096 1月  22 12:51 rpc1
drwxr-xr-x 2 kevinwu kevinwu 4096 1月  22 12:51 rpc2
-rw-r--r-- 1 kevinwu kevinwu  161 1月  22 12:51 rpccallback.go
drwxr-xr-x 2 kevinwu kevinwu 4096 1月  22 12:51 rpccommon
-rw-r--r-- 1 kevinwu kevinwu  138 1月  22 12:51 server.go
drwxr-xr-x 2 kevinwu kevinwu 4096 1月  22 12:51 test

service/api:
总用量 48
-rw-r--r-- 1 kevinwu kevinwu 10223 1月  22 12:51 conversions.go
-rw-r--r-- 1 kevinwu kevinwu 10737 1月  22 12:51 prettyprint.go
-rw-r--r-- 1 kevinwu kevinwu  1728 1月  22 12:51 prettyprint_test.go
-rw-r--r-- 1 kevinwu kevinwu 19189 1月  22 12:51 types.go

service/dap:
总用量 172
...
```

* 示例五：显示列表形式的文件详情，排除某些特定文件
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls -al -I appveyor.yml
总用量 136
drwxr-xr-x 13 kevinwu kevinwu  4096 1月  22 12:51 .
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 ..
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 assets
-rw-r--r--  1 kevinwu kevinwu 28773 1月  22 12:51 CHANGELOG.md
-rw-r--r--  1 kevinwu kevinwu   173 1月  22 12:51 .cirrus.yml
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 cmd
-rw-r--r--  1 kevinwu kevinwu  2440 1月  22 12:51 CONTRIBUTING.md
-rw-r--r--  1 kevinwu kevinwu   239 1月  22 12:51 .deepsource.toml
drwxr-xr-x  7 kevinwu kevinwu  4096 1月  22 12:51 Documentation
drwxr-xr-x 13 kevinwu kevinwu  4096 1月  22 12:51 _fixtures
drwxr-xr-x  8 kevinwu kevinwu  4096 1月  22 12:51 .git
-rw-r--r--  1 kevinwu kevinwu    29 1月  22 12:51 .gitattributes
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 .github
-rw-r--r--  1 kevinwu kevinwu   105 1月  22 12:51 .gitignore
-rw-r--r--  1 kevinwu kevinwu   863 4月   3 21:07 go.mod
-rw-r--r--  1 kevinwu kevinwu  7389 1月  22 12:51 go.sum
-rw-r--r--  1 kevinwu kevinwu   420 1月  22 12:51 ISSUE_TEMPLATE.md
-rw-r--r--  1 kevinwu kevinwu  1079 1月  22 12:51 LICENSE
-rw-r--r--  1 kevinwu kevinwu   565 1月  22 12:51 Makefile
drwxr-xr-x 12 kevinwu kevinwu  4096 1月  22 12:51 pkg
-rw-r--r--  1 kevinwu kevinwu  2152 1月  22 12:51 README.md
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 _scripts
drwxr-xr-x  9 kevinwu kevinwu  4096 1月  22 12:51 service
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 .teamcity
-rw-r--r--  1 kevinwu kevinwu  1405 1月  22 12:51 .travis.yml
drwxr-xr-x  6 kevinwu kevinwu  4096 1月  22 12:51 vendor
```

* 示例六：按修改时间排序，显示列表形式的文件详情
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls -lt
总用量 100
-rw-r--r--  1 kevinwu kevinwu   863 4月   3 21:07 go.mod
drwxr-xr-x  6 kevinwu kevinwu  4096 1月  22 12:51 vendor
drwxr-xr-x  9 kevinwu kevinwu  4096 1月  22 12:51 service
drwxr-xr-x 12 kevinwu kevinwu  4096 1月  22 12:51 pkg
drwxr-xr-x  3 kevinwu kevinwu  4096 1月  22 12:51 cmd
-rw-r--r--  1 kevinwu kevinwu  7389 1月  22 12:51 go.sum
-rw-r--r--  1 kevinwu kevinwu  1052 1月  22 12:51 appveyor.yml
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 assets
drwxr-xr-x 13 kevinwu kevinwu  4096 1月  22 12:51 _fixtures
drwxr-xr-x  2 kevinwu kevinwu  4096 1月  22 12:51 _scripts
drwxr-xr-x  7 kevinwu kevinwu  4096 1月  22 12:51 Documentation
-rw-r--r--  1 kevinwu kevinwu   420 1月  22 12:51 ISSUE_TEMPLATE.md
-rw-r--r--  1 kevinwu kevinwu  1079 1月  22 12:51 LICENSE
-rw-r--r--  1 kevinwu kevinwu   565 1月  22 12:51 Makefile
-rw-r--r--  1 kevinwu kevinwu  2152 1月  22 12:51 README.md
-rw-r--r--  1 kevinwu kevinwu 28773 1月  22 12:51 CHANGELOG.md
-rw-r--r--  1 kevinwu kevinwu  2440 1月  22 12:51 CONTRIBUTING.md
```

* 示例七：显示列表形式的文件详情，其中文件大小阅读友好的形式
```bash
kevinwu@debian:~/go/src/github.com/go-delve/delve$ ls -lh
总用量 100K
-rw-r--r--  1 kevinwu kevinwu 1.1K 1月  22 12:51 appveyor.yml
drwxr-xr-x  2 kevinwu kevinwu 4.0K 1月  22 12:51 assets
-rw-r--r--  1 kevinwu kevinwu  29K 1月  22 12:51 CHANGELOG.md
drwxr-xr-x  3 kevinwu kevinwu 4.0K 1月  22 12:51 cmd
-rw-r--r--  1 kevinwu kevinwu 2.4K 1月  22 12:51 CONTRIBUTING.md
drwxr-xr-x  7 kevinwu kevinwu 4.0K 1月  22 12:51 Documentation
drwxr-xr-x 13 kevinwu kevinwu 4.0K 1月  22 12:51 _fixtures
-rw-r--r--  1 kevinwu kevinwu  863 4月   3 21:07 go.mod
-rw-r--r--  1 kevinwu kevinwu 7.3K 1月  22 12:51 go.sum
-rw-r--r--  1 kevinwu kevinwu  420 1月  22 12:51 ISSUE_TEMPLATE.md
-rw-r--r--  1 kevinwu kevinwu 1.1K 1月  22 12:51 LICENSE
-rw-r--r--  1 kevinwu kevinwu  565 1月  22 12:51 Makefile
drwxr-xr-x 12 kevinwu kevinwu 4.0K 1月  22 12:51 pkg
-rw-r--r--  1 kevinwu kevinwu 2.2K 1月  22 12:51 README.md
drwxr-xr-x  2 kevinwu kevinwu 4.0K 1月  22 12:51 _scripts
drwxr-xr-x  9 kevinwu kevinwu 4.0K 1月  22 12:51 service
drwxr-xr-x  6 kevinwu kevinwu 4.0K 1月  22 12:51 vendor
```