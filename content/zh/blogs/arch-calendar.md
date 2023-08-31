---
title: 聊一聊日历的系统设计（上）
toc: true
authors:
  - kevinwu
tags:
  - architecture
series:
  - 架构设计精粹
date: '2023-08-25'
lastmod: '2023-08-25'
draft: false
---

## 前言
因为一些原因，我已不再从事飞书音视频业务，目前主要以技术管理者的角色负责飞书日历业务。实际上，这个岗位变动已经持续近一年的时间，这段时间也让我对日历类型的产品和架构产生了不一样的理解。因此，本文我想整体聊一聊日历的系统设计。

说到日历，很多人的第一反应可能是类似于下图这样的中国年历。年历的特点是**按照年-月-日的组织维度去展示日期**。一般来说，年历对于我们生活上而言主要是用于查阅，例如：今天星期几、今天对应农历的几月初几、今天是否节假日等等。
![中国2023年年历](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308281111727.png)

除此之外，有些人会在日历上面记录自己的个人待办事项。这使得日历的用法更加灵活，能够发挥更大的作用。
![待办事项](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308282011585.png)

总得来说，随着现在软件的发展，日历也呈现出更加现代化的产品形态。所谓日历，**本质上是一种时间容器**，与时间属性相关的内容都可以通过日历的形态去承载。**这里也打个广告，感兴趣的同学可以了解一下[飞书](https://www.feishu.cn/)和[飞书日历](https://www.feishu.cn/product/calendar)。**
![飞书日历](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308282108314.png)

## <font color="#FF0000">重要声明</font>
<font color="#FF0000"> **本文内容与飞书日历的设计现状并无任何关系，仅仅是代表个人的理解！飞书日历的系统设计由于合规和历史技术债的原因，是比较复杂的。** </font>

## RFC5545 && ICS
来思考一个问题：同一份日历能否在不同的日历系统（如：Microsoft Outlook、Google Calendar、Apple Calendar等等）之间交互呢？举个例子：假如我在公司日常工作中主要使用Outlook来记录我的工作日程，但是下班之后我也想在个人的手机端，苹果日历上面也能够看到这些内容，要怎么做呢？

考虑到以上的场景，IETF发布了 **[RFC5545](https://datatracker.ietf.org/doc/html/rfc5545)：Internet Calendaring and Scheduling Core Object Specification (iCalendar)**，它定义了通用日历的核心概念。

<iframe height=850 width=100% src="https://www.processon.com/view/link/59e2d497e4b040dc85064620#map" frameborder=0  allowfullscreen> </iframe>

**ICS**文件则是基于这个标准而创建的，它是一种纯文本的文件格式，能够被不同的日历系统直接打开或者是导入。ICS文件示例如下：
```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Example Inc.//Example Calendar//EN
BEGIN:VEVENT
UID:123456@example.com
DTSTAMP:20220101T120000Z
ORGANIZER;CN=John Doe:mailto:john.doe@example.com
DTSTART:20220110T140000Z
DTEND:20220110T150000Z
SUMMARY:Important Meeting
LOCATION:Conference Room A
DESCRIPTION:This is an important meeting to discuss the project status.
ATTENDEE;RSVP=TRUE;CUTYPE=INDIVIDUAL;CN=Jane Smith:mailto:jane.smith@example.com
END:VEVENT
END:VCALENDAR

这个ICS文件包含了一个日历事件，事件的相关信息如下：
- 事件唯一标识符（UID）：123456@example.com
- 时间戳（DTSTAMP）：2022年1月1日 12:00:00（UTC时间）
- 组织者（ORGANIZER）：John Doe，邮箱为john.doe@example.com
- 事件开始时间（DTSTART）：2022年1月10日 14:00:00（UTC时间）
- 事件结束时间（DTEND）：2022年1月10日 15:00:00（UTC时间）
- 事件概要（SUMMARY）：Important Meeting
- 事件地点（LOCATION）：Conference Room A
- 事件描述（DESCRIPTION）：这是一个重要的会议，用于讨论项目状态。
- 参与者（ATTENDEE）：Jane Smith，需要回复（RSVP=TRUE），邮箱为jane.smith@example.com
```

ICS如同JSON/XML，是不同日历系统之间沟通的统一“语言”：
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308291543161.png)

## 核心概念
广义上，我们提及的日历是指围绕着这种特定类型的业务或产品，包括在日历上面记录的一系列活动或事件。狭义上，在日历系统这个特定的上下文中，日历其实包括了2个核心概念：日历和日程。
* **日历（Calendar）**：特指一系列日程的集合体。
* **日程（Event）**：描述的是具有起始时间和结束时间的活动或事件。
![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308291615383.png)

在下文中，我会注意区分这两个概念的用词。

### 日历
#### 日历的主体
日历的主体，是指可以对日历进行操作的对象。通常来说，在日历类型的产品中，主体指代的是**用户（User）**。用户可以创建一个或者多个日历，也可以订阅来自其它用户所创建的日历。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308291707858.png)

更进一步来看，日历的主体可以是**任意的实体**。例如：会议室、棋牌室、培训室、轰趴馆、教室、医院等等，它们同样可以作为日历的主体。以会议室为例，我们经常希望能够查阅到会议室的安排情况，以便于预定会议室的空闲时间段。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308291732047.png)

再进一步来看，日历主体对日历的操作也**不限于创建和订阅**。例如：老板和助理之间，可以由助理**代管**老板的日历。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308291749248.png)

#### 日历的类型
在RFC5545中，日历实则并没有定义什么类型。但通常在日历产品中，根据不同的业务场景，日历的用途不尽相同。例如：
* **主日历**：日历产品通常会为每个注册用户默认创建一个日历，这里命名为主日历。当我们要创建日程去邀请其它用户作为参与人的时候，实际上就是邀请对方的主日历。主日历的生命周期建议与用户相同，一般不可被删除，除非用户信息被注销。
* **共享日历**：共享日历偏向于协作性质的日历，可用于团队项目类的协同场景。共享日历的特点是可以允许被不同的用户查看、编辑、删除、更新上面的日程，从而达到合作的目的。
* **只读日历**：只读日历适用于公告类的日历，例如：公司官方通告日历、节假日日历等。只读日历并非是完全只读，而是指其通常不归属于系统用户，对系统用户来说是只读状态。只读日历可以被类似于系统管理员的身份去编辑和修改。

### 日程
#### 单次日程 && 重复性日程
日程是日历系统中的核心，相比于日历来说更复杂得多。根据是否重复的维度，日程可以划分为单次日程和重复性日程。所谓重复性日程就是指按照一定周期性重复出现的日程。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308301522824.png)

重复性日程可以说是导致日历系统复杂的核心源头，几乎所有在单次日程中的业务场景，扩展到重复性日程中都需要好好设计。

#### 重复性规则
RFC5545中通过RRULE来定义重复性规则，它基于一系列的属性来描述：
1. FREQ（频率）：表示事件重复的频率，如：DAILY（每天）、WEEKLY（每周）、MONTHLY（每月）或YEARLY（每年）等。
2. INTERVAL（间隔）：表示在给定频率上重复事件之间的间隔数量。例如，FREQ=WEEKLY 且 INTERVAL=2，表示事件每两周发生一次。
3. COUNT（次数）：表示事件的重复次数。例如，FREQ=DAILY 且 COUNT=10，表示事件连续10天发生一次。
4. UNTIL（截止日期）：表示一个终止日期，事件不再重复发生。例如，FREQ=DAILY 且 UNTIL=20211231，表示事件每天重复，直到2021年12月31日为止。
5. BYDAY（按天）：表示事件在周几重复。例如，FREQ=WEEKLY 且 BYDAY=MO,WE,FR，表示事件在周一、周三和周五重复。
6. BYMONTHDAY（按月的某天）：表示事件在月份中的哪些日期重复。例如，FREQ=MONTHLY 且 BYMONTHDAY=1,15，表示事件在每月的1号和15号重复。
7. BYYEARDAY（按年的某天）：表示事件在一年中的哪些天重复。例如，FREQ=YEARLY 且 BYYEARDAY=1,365，表示事件在每年的第一天和最后一天重复。
8. BYWEEKNO（按周数）：表示事件在一年中的哪些周重复。例如，FREQ=YEARLY 且 BYWEEKNO=1,52，表示事件在每年的第一周和最后一周重复。
9. BYMONTH（按月份）：表示事件在一年中的哪些月份重复。例如，FREQ=YEARLY 且 BYMONTH=1,12，表示事件在每年的1月和12月重复。
10. BYSETPOS（按位置）：表示在生成的事件序列中选择特定位置的事件。例如，FREQ=MONTHLY 且 BYDAY=MO,TU,WE,TH,FR 且 BYSETPOS=-1，表示每月的最后一个工作日发生事件。

RRULE可以通过组合这些属性来创建复杂的重复事件模式。

假设我们有一个每周一和周四重复的日程事件，从2022年1月1日开始，持续10次。iCalendar表示如下：
```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Example Corp.//Example Calendar//EN
BEGIN:VEVENT
UID:1@example.com
DTSTAMP:20220101T000000Z
DTSTART:20220101T100000
DTEND:20220101T110000
SUMMARY:重复性日程事件
RRULE:FREQ=WEEKLY;COUNT=10;BYDAY=MO,TH
END:VEVENT
END:VCALENDAR
```

#### 日程 && 实例
当一个日程具有重复性质，那么就产生了两层概念：第一层是日程本身的定义和重复性规则，第二层是每一次重复的具体内容，称之为**实例（Instance）**。而对于单次日程来说，日程和实例指代的是同一份内容。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308301553411.png)

需要特别强调的一点是：在RFC5545中，ICS文件中并没有所谓Instance这个概念，但是在系统设计和实现的过程中，我们则需要注意Event和Instance之间的区别和关联。

#### 例外日程
当我们修改重复性日程某个特定实例的属性，例如起始时间和结束时间。此时，该实例就变成了一个**例外（Exception）**。例外如同单次日程，它同样有两层含义，第一层是例外日程，第二层是例外实例。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308301655295.png)

不仅如此，由于例外日程诞生自原重复性日程，因此例外日程和重复性日程需要有所关联。在RFC5545中，例外日程和原重复性日程拥有相同的**UID**，同时例外日程通过**RECURRENCE-ID**来作为唯一标识，RECURRENCE-ID是与实例原始时间有关的一个属性，它不会因为后续该实例起始时间和结束时间的变化而变化。

举个具体例子：
假设有一个每周一次的会议，从1月1日开始，共4次。现在，你想要为1月15日的实例更改开始时间，那么你可以使用RECURRENCE-ID属性来表示这个特定的实例。以下是一个iCalendar文件的例子，其中包含一个每周一次的会议事件，以及一个修改后的实例，其中RECURRENCE-ID属性用于标识1月15日的实例：

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Example Corp.//Example Calendar//EN
BEGIN:VEVENT
UID:12345@example.com
DTSTAMP:20211201T090000Z
DTSTART;TZID=America/New_York:20220101T100000
DTEND;TZID=America/New_York:20220101T110000
RRULE:FREQ=WEEKLY;COUNT=4
SUMMARY:Weekly Meeting
END:VEVENT
BEGIN:VEVENT
UID:12345@example.com
RECURRENCE-ID;TZID=America/New_York:20220115T100000
DTSTAMP:20211201T090000Z
DTSTART;TZID=America/New_York:20220115T110000
DTEND;TZID=America/New_York:20220115T120000
SUMMARY:Weekly Meeting - Time Change
END:VEVENT
END:VCALENDAR
```

在这个例子中，RECURRENCE-ID属性 (`RECURRENCE-ID;TZID=America/New_York:20220115T100000`) 用于表示1月15日的会议实例。然后，你可以为这个特定的实例修改开始和结束时间，而不影响其他循环实例。

#### 后续日程
主流的日历产品中，通常有3种重复性日程的编辑动作：
* 编辑此次
* 编辑所有
* 编辑此次及后续

针对以上三种编辑动作，RFC5545只针对“编辑此次”有比较明确的定义，即上一小节提及的例外。而针对“编辑所有”和“编辑此次及后续”，**这取决于不同日历产品的具体实现**。但不管产品如何定义，“编辑所有”和“编辑此次及后续”应当保持相同的行为准则，差异在于**作用域**不同。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308311726385.png)

**“编辑此次及后续”动作完成之后，将产生一个全新的重复性日程，它具有独立的UID。**

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308311730240.png)

#### 日程参与人
日程的参与人是日程的核心元素。**抽象来说，日程参与人可以连接不同主体的日历，乃至不同系统的不同主体的日历。**

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-arch-calendar/202308311503212.png)

在RFC5545中，参与人通过**ATTENDEE**属性来描述。以下是一个ATTENDEE属性的示例：

`ATTENDEE;ROLE=REQ-PARTICIPANT;PARTSTAT=TENTATIVE;CN=John Doe:mailto:john.doe@example.com`

在这个示例中，参与者的邮箱地址是"john.doe@example.com"，角色是"REQ-PARTICIPANT"（请求参与者），参与状态是"TENTATIVE"（暂定），并且参与者的名称（CN）是"John Doe"。

参与人也可以是一个邮件组。要表示一个邮件组，可以在ATTENDEE属性中使用一个表示邮件组的URI。这通常是一个"mailto" URI，其中包含邮件组的电子邮件地址。例如：

`ATTENDEE;CN=Team A:mailto:teama@example.com`

在这个示例中，参与者是一个名为"Team A"的邮件组，其电子邮件地址是"teama@example.com"。

然而，请注意，并非所有日历客户端都支持邮件组作为参与者。在实际应用中，你可能需要将邮件组展开为各个成员，并将它们作为单独的ATTENDEE属性添加到日程中。这可以确保更广泛的兼容性和与各种日历客户端的正确交互。

## 参考
1. [RFC5545](https://datatracker.ietf.org/doc/html/rfc5545)

## 总结
本文是日历系统设计的上篇，主要讲解日历的互联网标准RFC5545，以及日历的两大核心概念：日历和日程。日程的重复性质，对整个日历业务带来了复杂的理解和挑战，也是技术实现的核心难点。在下篇中，我将会结合个人经验，重点讲解一下日历系统设计中的一些难点和思路。