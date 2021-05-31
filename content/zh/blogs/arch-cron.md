---
title: 架构设计精粹之CronJob
toc: true
authors:
  - kevinwu
tags:
  - architecture
date: '2021-04-26'
lastmod: '2021-04-27'
draft: false
---

定时任务**CronJob**是系统设计中的基础原料，它提供了一种通用的后台任务解决方案。本文将由浅入深讲解CronJob任务调度架构设计中的关键要素。

## 使用场景
CronJob广泛用于定期执行的后台异步任务，适合CronJob的业务场景有：

1. 机器/服务性能指标的定时上报。如：每10s上报一次机器CPU利用率，用于监控系统。
2. 数据报表的每日生成。如：每天10点触发一次离线任务，完成数据清洗和数据聚合，展示业务DAU。
3. 过期数据的定时清理。如：每10分钟触发一次数据扫描，计算出已经过期的数据并删除。
4. ...

等等诸如此类满足**定时**和**异步**这两个特性的业务场景，均可使用CronJob来完成。进一步来说，CronJob的能力可以被集中到一个**CronJob任务调度中心**，完成对定时任务的通用管理和分发。

## 基本形态
CronJob的基本形态：定时周期+任务。

### 定时周期
[**Cron Expressions**](https://en.wikipedia.org/wiki/Cron)是一种定时周期描述字符串，它通常包括6个由空格隔开的字段。它们分别代表：

| **字段**               | **取值范围**       | **特殊字符**                | **备注**                |
|----------------------|----------------|-------------------------|-----------------------|
| Seconds（秒）           | 0-59           | `*` `,` `-`             |                       |
| Minutes（分）           | 0-59           | `*` `,` `-`             |                       |
| Hours（时）             | 0-23           | `*` `,` `-`             |                       |
| Day of month（每月的第几天） | 1-31           | `*` `,` `-` `?` `L` `W` | `?` `L` `W` 只在某些系统中支持 |
| Month（月）             | 1-12 或 JAN-DEC | `*` `,` `-`             |                       |
| Day of week（每周的第几天）  | 0-6 或 SUN-SAT  | `*` `,` `-` `?` `L` `#` | `?` `L` `#` 只在某些系统中支持 |

标准特殊字符的含义：
1. `*` 匹配任意值。例如：Seconds填`*`会在符合要求的每秒钟都触发任务。
2. `,` 用于分隔不同的取值。例如：Seconds填`1,2,5,7`会在符合要求的第1、第2、第5、第7秒钟分别触发任务。
3. `-` 用于一段连续取值。例如：Seconds填`1-7`会在符合要求的第1~7秒钟分别触发任务。

非标准特殊字符的含义：
1. `?` 仅用在Day of month和Day of week。由CronJob启动时间来决定`?`的取值。例如：该CronJob是在星期六启动，`* * * * * ？`则会变成`* * * * * 6`。
2. `L` 仅用在Day of month和Day of week。`L`在Day of month中表示当月的最后一天，`L`在Day of week中可以用`5L`表示该月中的最后一个星期五。
3. `W` 仅用在Day of month。主要用于工作日触发。例如：`15W`表示最接近当月第15天那个工作日触发。如果第15天是星期六，就第14天星期五触发；如果第15天是星期日，就第16天星期一触发。
4. `#` 仅用在Day of week。`5#3`表示每个月的第3个星期五。

光看定义不容易理解，下面举例说明Cron表达式的含义：

1. `* * * * * *`：每秒钟触发。
2. `0 * * * * *`：每分钟的第0秒触发。
3. `0 3,15 * * * *`：每小时的第3、15分触发。
4. `0 3,15 8-11 * * 1`：每周一上午8到11点的第3和第15分钟触发。
5. `0 45 4 1,10,22 * *`：每月1、10、22日的4点45分触发。
6. `*/5 * * * * *`：每5秒钟触发。
7. ...

### 任务
任务本身的定义是由CronJob的使用方来决定，通常属于业务逻辑范畴。但CronJob任务调度中有一点非常重要：**任务独占**。所谓的任务独占有两个重要维度：

1. 同一个任务未完成时候，下一次定时周期触发直接跳过。
2. 同一个任务只允许一个实例执行，不允许多个实例并发执行。

任务独占对于设计一个高可用的CronJob任务调度中心来说需要重点考虑！

1. 解决维度1，我们可以为每个任务生成ID和Status，当任务执行时候将Status设置为Busy，每次任务执行之前检测相同ID的任务是否已经是Busy状态来跳过任务。
2. 解决维度2，我们需要保证即使存在多个CronJob任务调度触发器，也只有一个是真正生效的。

## 架构设计
让我们由小场景到大场景，从演进角度来讲解一个CronJob任务调度的架构设计。

### 单实例不解耦
假如业务本身是单实例的，我们当然可以直接在业务服务的后台运行一个CronJob调度器，这是最简单的业务结构：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-1.png)

### 多实例不解耦
但通常，业务服务在生产环境中都会采用多实例部署，防止单点故障：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-2.png)

这时，不同业务服务实例会运行不同的CronJob调度器，这适用于任务不需要独占的场景。而对于任务需要独占的场景则无法解决，则要考虑引入分布式锁等中间件。

### CronJob调度器解耦
换个方向考虑，我们将CronJob调度器独立出来成为一个单独的服务并单实例部署。这种架构也能够解决任务独占的场景：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-3.png)

但这种架构的缺点在于：
* CronJob调度器的升级过程难以协调。升级过程中如果先停后启（先停止旧实例再升级新实例），那么停止过程中的CronJob调度器也停止工作了。升级过程中如果先启后停（先启动新实例再停止旧实例），虽然CronJob调度器持续工作，但会在过渡期同时存在两个CronJob调度器同时工作而导致冲突。
* CronJob调度器本身是单点部署的，不适合在需要高可用的生产环境中落地。

### 高可用CronJob调度器解耦
因此考虑CronJob调度器本身也需要高可用，这里面有2个思路：分布式锁和分布式选主。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-4.png)

至此，一个高可用的CronJob任务调度框架基本完成。我们可以围绕这个框架的核心要素来进一步完善各种高级特性。

## 高级特性
### 管理后台
一个功能齐全的CronJob任务调度框架通常会包括可视化的管理后台。管理后台主要提供：

1. CronJob任务的列表展示、创建、修改和删除。
2. CronJob任务的运行历史、日志等信息。
3. 参与CronJob框架节点IP列表和主节点等信息。
4. ...

因此，为了进一步完善CronJob调度器的管理后台能力，我们可以抽象并独立出Cron API Server。此时，CronJob任务调度框架的架构大致如下：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-5.png)

由UI/API/DB组成Cron Admin的基本功能，CronJob调度器集群的Master将会从API Server中获取需要定时运行的任务并开启时钟运行。运行任务的具体内容则是通过调用业务服务的API Gateway去完成，通常API Gateway会内置负载均衡的功能，将最终的任务发往业务服务执行。

### 任务分片
任务分片是CronJob调度框架提升任务吞吐的优化措施，任务分片需要业务服务本身配合来完成。由于任务的具体内容其实是业务服务自定义的，因此若业务方能够根据自身特点提供一种合理的方式来划分任务执行粒度，将不同的任务分割到不同的切片当中。

对于能够定义出分片key的任务，CronJob调度框架可以将任务切分并发往API Gateway，再由API Gateway转发给业务服务执行：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-6.png)

### 调度策略
架构拆分至此，我们可以发现一个关键点：任务调度本质上是由API Gateway代为完成的，但通常API Gateway只会考虑的负载均衡，不会考虑更加复杂的调度策略。因此，如果CronJob调度框架希望在这个功能点上面发力的话，可以进一步完善注册中心。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-7.png)

业务服务将节点信息上报给注册中心，这样一来CronJob调度器Master就能够获取到节点列表，进一步可以定制各种调度策略：

1. RoundRobin
2. VIP节点优先
3. 业务节点分组
4. ...

### 失败重试
失败重试可以从两个角度去完善：

1. 业务服务本身执行任务失败的时候，可以尝试重试。
2. 业务服务本身执行任务失败的时候，可以有机制通知CronJob调度器Master，再由调度器重新选择其它节点重试。

因此，我们可以抽象出一个Job SDK嵌入到业务服务。将包括注册、Job Executor接口定义、重试机制等等逻辑进一步封装，并提供统一的接入方式。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-cron/cron-8.png)

## 参考
1. [Cron维基百科](https://en.wikipedia.org/wiki/Cron)
2. [xxl-job](https://github.com/xuxueli/xxl-job)
3. [cronsun](https://github.com/shunfei/cronsun)

## 总结
本文介绍了CronJob的使用场景和基本形态，并由浅入深详解CronJob任务调度架构设计的关键要素和高级特性扩展。