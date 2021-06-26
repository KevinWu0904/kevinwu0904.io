---
title: 深入解析Go Modules
toc: true
authors:
  - kevinwu
tags:
  - golang
series:
  - 深入理解Go语言
date: '2021-04-16'
lastmod: '2021-04-16'
draft: false
---

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-common/golang-logo.png)

Go语言的依赖管理经历了漫长的迭代和演进，最终随着[Go Modules](https://golang.org/ref/mod)被官方采纳，形成大一统局面。回顾整个历史，Go依赖管理的实践之路其实颇为坎坷，中途甚至还曾引发Go官方团队和Go社区团队之间的纠纷。但从结果来看，Go Modules终结了Go语言依赖管理的乱象，对于广大Go开发者来说是个福音。

## Go依赖管理的演进历史
### 阶段一：Makefile、goinstall、go get
早在2009年11月，Go最初版本的发布只包括：编译器、连接器和一些基本的库。此时Go程序需要自己运行`6g`和`6l`完成编译过程，因此官方发布的程序中也附加了一些Makefile的示例。

2010年2月，官方引入goinstall（注意：这里的goinstall和go命令中的go install不是同一个概念）。这是一个零配置的命令，从代码仓库中直接下载包。goinstall在当时引入了现在开发者已经非常熟悉的包导入规则，后面逐步形成明朗的生态系统。goinstall终结了Makefile中五花八门实现造成的复杂性，这种化简对于形成统一工具链也具有极大正面作用。

2011年12月，go get代替goinstall进入go命令，并成为Go官方支持的工具链。总体而言，go get相比于C/C++的依赖管理来说有革命性进步：
1. 允许Go的开发者方便地共享代码，和依赖他人的包进行构建。
2. 将构建系统的细节隔离在go命令中。

但go get缺少版本管理功能，当go get请求一个包的时候，它总是拉取最新的代码，并将拉取动作交给Git、Mercurial等版本管理系统去做。go get对于版本管理的缺少至少带来两个致命问题。

### 阶段二：语义版本和API稳定性
go get的第一个致命问题：由于没有版本的概念，在包更新时候它不知道该以什么方式通知用户（兼容性的还是非兼容性的）。

2013年11月，Go 1.2的FAQ中增加了一条关于包版本管理的基本建议（此建议直到Go 1.10都没变过）：
> 发布到公共使用的包，在演进时应尽量保持向后兼容性（如v1.5应兼容v1.4）。[Go 1的兼容性指南](https://golang.org/doc/go1compat.html)就是一个很好的参考：不要移除已暴露的命名，鼓励使用组合单词来命名，等等。若需不同的功能，请创建一个新的命名，而非直接修改旧名字下的功能。若必须引入不兼容的修改，请创建一个新的包，并使用新的`import`路径。

2014年3月，Gustavo Niemeyer创建了[gopkg.in](https://gopkg.in/)，倡议在Go语言中编写稳定的API。此网站可从`go get`的URL中解释出包的版本，并转发 GitHub中对应版本的源码，如`gopkg.in/yaml.v1`、`gopkg.in/yaml.v2`分别对应Git仓库中的不同提交（也可能刚好是不同的分支）。基于语义的版本管理（semver），gopkg希望包作者在引入不兼容性修改时创建一个新的主版本号，由于`v2`是完全不同的API，所以`v2`的`import`路径可逐步替换`v1`的`import`路径（注：因为二者可被同时使用）。

### 阶段三：Vendor模式和可复现构建
go get的第二个致命问题：它无法保证构建是稳定可复现的，这意味着包的开发者和包的使用者很容易出现依赖不同版本的情况。

2013年11月，Go 1.2的FAQ中还添加了这条基本建议：
> 使用一个外部的依赖包时，你若担心该包无故发生变化，最简单粗暴的解决办法是复制一份到你的本地仓库（Google内部也是使用这种方法）。将副本存储在一个新的`import`路径中，将其标识为本地副本。例如，你可将`original.com/pkg`复制为`you.com/external/original.com/pkg`。Keith Rarick 写了一个名为`goven`的工具将此过程自动化。

在这个阶段，各种第三方的Go依赖管理工具已经开始出现。它们都基于复制依赖的原理，通过非官方Hack的方式来完成可复现构建。

因此，Go官方在看到这个乱象之后，正式推出官方vendor机制。vendor目录用于存放依赖包，加载顺序是：vendor > GOROOT > GOPATH。vendor于Go 1.5进入实验阶段，于Go 1.6成为默认启动的功能。

从本质上说，vendor机制只是包管理的一种不完整解决方案。它只提供了稳定可复现构建的能力，但并没有告诉项目具体用哪个版本。于是类似glide、godep、govendor等第三方包管理工具，通过添加特殊的版本信息记录文件，来做隐式包版本管理。但由于Go的官方工具链都无法识别类似的文件，因此导致Go工具链生态与之割裂。

### 阶段四：官方包管理器实验
在 GopherCon 2016 会议之后，Go成立了一个包依赖管理器委员会和一个讨论小组，目的是创造一个新的依赖管理工具[dep](https://github.com/golang/dep)。这个工具的愿景是统一目前各种Go的包管理器，继续使用vendor目录，但依然不属于Go官方工具链的原生功能。

引用Russ Cox在博客中所说：
> dep有几大作用：
> * 它是今天为止可用的工具实践中的重要进步
> * 它是朝一个解决方案前进的重要一步
> * 它也是“官方实验”，试验哪些功能对Go开发者有用或没用
> 
> 但dep不是go命令中整合版本管理的直接原型。dep是一个强大的、很灵活的工具，探索了包管理的设计空间，在我们争论着如何对Go程序进行构建时，扮演着类似Makefile的角色。但当我们对包管理的设计空间了解得更深、明白到哪些关键特性必须实现后，我们才知道如何在Go生态系统中移除那些不必要的功能、灵活性。然后采取强制的约定，使得Go的代码库更统一、更易于理解、相关工具更易于构建。

### 阶段五：vgo提案和最终的官方解决方案Go Modules
dep终究没有被官方采纳，虽然Russ Cox肯定了dep的历史作用，但dep本质上仍是vendor机制下的延伸，并且dep坚持语义版本，而并未使用Go Modules方案中的语义导入版本。因此，vgo提案正式提出。

vgo由Russ Cox于2018年提出，它建议采用语义导入版本规则结合最小版本选择规则。另一方面，vgo希望能够与Go官方工具链相结合，因此最初以“vgo”命名，并且实验性阶段以替代go命令的方式运行。

vgo很快得到官方认可，并作为正式proposal开发合入主干，最终以go mod命令正式纳入官方工具链。

Go Modules去除GOPATH和vendor目录的依赖，不再需要基于复制依赖的做法，大大减少源码包的体积并杜绝了修改vendor目录内容的行为。另一方面，Go Modules还集成进了go命令中，配合go get、go list、go build等等命令协同工作，整体体验更加优秀。

至此，Go结束了漫长的依赖管理之争，最终形成大一统局面。但另一方面，vgo的发布引发了社区的强烈动荡，以dep为首的开发者为此感到沮丧[^1]。

## Go Modules
### 语义版本vs语义导入版本
**语义版本**（semantic versioning）是指在Go语言中对同一个依赖的不同兼容性版本使用统一的import名称，在辅助文件中记录版本信息。语义版本的代表是dep，例如：
```go
import (
	"github.com/robfig/cron"
)
```

```toml
# Gopkg.toml 版本1.2.0
[[constraint]]
  name = "github.com/robfig/cron"
  version = "1.2.0"
  

# Gopkg.toml 版本3.0.1
[[constraint]]
  name = "github.com/robfig/cron"
  version = "3.0.1"
```

**语义导入版本**（semantic import versioning）则是指在Go语言中对同一个依赖的不同兼容性版本使用不同的import名称，同时也在辅助文件中记录版本信息。导入版本的代表是go module，例如：

* v1大版本
```go
import "github.com/robfig/cron" // v1大版本
```
```go
// go.mod
require (
	github.com/robfig/cron v1.2.0
)
```

* v3大版本
```go
import "github.com/robfig/cron/v3" // v3大版本
```
```go
// go.mod
require (
	github.com/robfig/cron v3.0.1
)
```

### import兼容性规则
Go语言的import兼容性规则是指：
> 如果一个旧包与新包共用相同的 import 路径，那么新包必须向后兼容旧包。

Go的开发者采用业界通用的semantic versioning作为版本标识符，它形如（vmajor.minor.patch）：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/semver.png)

Go的兼容性规则希望开发者遵守如下守则：
1. 相同大版本，新包需要完全兼容旧包（通过诸如不删除弃用方法，不修改导出变量等手段）。例如：v1.3.1需要完全兼容v1.2.0。
2. 不同大版本，允许存在不兼容改动。例如：v3.0.0和v1.0.0允许声明不兼容的接口方法。
3. v0.0.0到v1.0.0之间由于处于开发阶段，允许存在大量不兼容的行为，但需要开发者自己处理。

进一步来说，go mod和dep核心的一个本质区别是：对同一个包的不同大版本，go mod在import路径中引入版本号，dep则始终保持保持不变的import路径。go mod这个非常巧妙的做法，能够完美解决go依赖管理中著名的“钻石型依赖”问题：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/diamond.png)

如上图所示：A直接依赖B和C，B依赖D的v1版本，C依赖D的v2版本，且D的这两个版本是不兼容的。这种情况下，无论是选择D v1还是D v2都无法得出正确的依赖构建，这便是钻石型依赖的难题。

那么go mod是如何解决这个问题的呢？

由于D v1和D v2属于D的不同大版本，因此当C引入D的时候，需要在import路径中加入`/v2`：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/resolve-diamond.png)

因此go mod最终产生的依赖同时包括A、B、C、D v1和D v2，而不像dep那样需要在D v1和D v2之前做一个取舍。

### 定义模块
go mod通过module（模块）来定义一组包的合集，它们最终拥有相同的版本号，而这组合集通常就是指一个Go的项目。例如：
```go
module k8s.io/kubernetes

go 1.15
```

因此上图中的package A/B/C，在go mod中更准确的表述应该是module A/B/C。

### 最小版本选择（MVS）
[**Minimal Version Selection**](https://research.swtch.com/vgo-mvs)是go mod所实现的关于依赖升级和降级的选择策略，它不同于Go之前的包管理器实践。

“最小版本选择”这个名称非常容易让人产生误解，对它的正确理解是：当模块的某个依赖发布更新的版本，MVS选择的结果并不一定会选择该版本，更有可能是保持不变。对它的常见**错误理解**是：当模块间接依赖同一个包的不同版本，比如v1.0.0和v1.0.1，选择v1.0.0。

接下来详细阐述最小版本选择的原理，以Russ Cox的博客示例作为讲解：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/mvs-1.png)

#### 算法一：BuildList
Go的依赖关系可以抽象成一张有向图，MVS首先要通过BuildList找到当前模块的完整构建视图。

这是一个BFS（广度优先遍历）算法：首先将A1入队，然后将A1直接依赖的B1.2和C1.2入队，重复这个过程，最终能够得到一个初始列表结果：[A1, B1.2, C1.2, D1.3, D1.4, E1.2]。需要注意一点，由于可能存在依赖环路，因此算法会记录依赖是否曾经入队，保证每个依赖只进队一次，防止无限循环。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/mvs-2.png)

接下来根据兼容性原则，D1.3和D1.4属于同一个大版本，因此D1.3和D1.4是兼容的。MVS会选择其中较新的版本，最终的列表L是：[A1, B1.2, C1.2, D1.4, E1.2]。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/mvs-3.png)

具体的源码分析可以参考：[https://github.com/golang/go/blob/release-branch.go1.15/src/cmd/go/internal/mvs/mvs.go](https://github.com/golang/go/blob/release-branch.go1.15/src/cmd/go/internal/mvs/mvs.go)。值得注意的是由于依赖的列表往往需要通过网络请求Github等代码仓库网站，因此整个BFS为了加速是并行执行的。

#### 算法二：Req
通过BuildList之后，我们得到了一个列表L：[A1, B1.2, C1.2, D1.4, E1.2]。但对于go mod来说，在确保能稳定可复现构建的前提下，它希望能够管理最少的依赖信息。Req算法就是用来完成这件事情，它能够计算出最少依赖列表。

Req的算法流程大致如下：

1. 遍历列表L，对于每个依赖进行DFS（深度优先遍历），将路径的结果记录进postorder数组。例如：
    * L：[A1, B1.2, C1.2, D1.4, E1.2]。
    * 遍历A1，由于A1是源点，直接排除。
    * 遍历B1.2，路径B1.2 -> D1.3 -> E1.2，由于DFS是基于栈结构（FILO，先入后出），postorder此时为[E1.2, D1.3, B1.2]。
    * 遍历C1.2，路径C1.2 -> D1.4 -> E1.2，同样方式处理，重复节点E1.2无须重复记录，postorder此时为[E1.2, D1.3, B1.2, D1.4, C1.2]。
    * 遍历D1.4和E1.2，路径D1.4 -> E1.2，路径E1.2出现的节点都已经重复。
    * 最终的postorder为[E1.2, D1.3, B1.2, D1.4, C1.2]。
2. 反向遍历postorder，记录min列表。对于每个节点n，首先判断是否属于该包同一个大版本中最新版本，如果不是则直接排除（比如D1.3和D1.4，同属于v1大版本，直接排除D1.3）。将n加入min数组，标记n能够访问的所有节点并排除。例如按照这个算法：
    * postorder为[E1.2, D1.3, B1.2, D1.4, C1.2]，注意是反向遍历。
    * 遍历C1.2，进入min列表，此时min[C1.2]。排除C1.2能够到达的D1.4和E1.2。
    * 遍历D1.4，已排除。
    * 遍历B1.2，进入min列表，此时min[C1.2，B1.2]。排除B1.2能够到达的D1.3和E1.2。
    * 遍历D1.3，D1.3不是D中最新版本，需要排除。
    * 遍历E1.2，已排除。
    * 最终的min为[C1.2，B1.2]。

#### 算法三：Upgrade
升级依赖是go mod一个核心场景，升级依赖的流程是：在有向图的基础上添加新的有向边，注意只是新增而不会删除旧的有向边。

例如A1当前依赖C1.2，但希望能够升级到C1.3：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/mvs-4.png)

算法流程是：
* 在A1和C1.3之间新增一条有向边。
* 通过算法一：BuildList，计算出列表L：[A1, B1.2, C1.3, D1.4, E1.2, F1.1, G1.1]。
* 通过算法二：Req，根据列表L，计算出min：[A1, B1.2, C1.3, D1.4]。

之所以是在A1和C1.3之间新增一条有向边，而没有删除A1和C1.2之间的有向边。是因为假如删除了A1和C1.2之间的有向边，那么计算出的结果会使用D1.3，而不是D1.4。这对于D来说是被动降级，不符合最小版本选择期望达到的效果。

#### 算法四：Downgrade
降级依赖是go mod另一个核心场景，不同于升级依赖是通过新增有向边而且不删除旧的有向边，降级依赖的流程是：删除非法依赖包和引用它的直接依赖，重新计算min。

例如A1间接依赖的D1.4发现BUG，实际上可能是D1.3就引入的，因此希望降级到D1.2：

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-golang-modules/mvs-5.png)

算法流程是：
* 标记D1.3、D1.4为不可用依赖。
* 将A1和D1.3、D1.4引用的E1.2新增一条有向边，防止E1.2被动降级。
* 找到依赖D1.3、D1.4的B和C的早期版本，A1和B1.1、C1.1分别新增一条有向边。
* 通过算法一：BuildList，计算列表L：[A1, B1.1, D1.1, E1.2, C1.1]。
* 通过算法二：Req，根据列表L，计算出min：[A1, B1.1, C1.1, E1.2]。

### go.mod和go.sum
go mod在工程中引入go.mod和go.sum文件，其中go.mod文件用于记录依赖的版本信息，go.sum文件用来记录依赖的hash值。

go.mod文件示例：

```go
module example.com/my/thing

go 1.12

require example.com/other/thing v1.0.2
require example.com/new/thing/v2 v2.3.4
exclude example.com/old/thing v1.2.3
replace example.com/bad/thing v1.4.5 => example.com/good/thing v1.4.5
retract [v1.9.0, v1.9.5]
```

* require：require记录的内容也即根据MVS中算法二：Req生成的列表。
* exclude：exclude可以记录一个版本的黑名单，防止该版本被引用。
* replace：replace可以用于替换一个依赖。
* retract：go1.16新特性，可以用于包的开发者紧急撤回某个已知BUG的版本。

go.sum文件示例：
```go
bitbucket.org/bertimus9/systemstat v0.0.0-20180207000608-0eeff89b0690 h1:N9r8OBSXAgEUfho3SQtZLY8zo6E1OdOMvelvP22aVFc=
bitbucket.org/bertimus9/systemstat v0.0.0-20180207000608-0eeff89b0690/go.mod h1:Ulb78X89vxKYgdL24HMTiXYHlyHEvruOj1ZPlqeNEZM=
cloud.google.com/go v0.54.0 h1:3ithwDMr7/3vpAMXiH+ZQnYbuIsh+OPhUPMFC9enmn0=
cloud.google.com/go v0.54.0/go.mod h1:1rq2OEkV3YMf6n/9ZvGWI3GWw0VoqH/1x2nd8Is/bPc=
cloud.google.com/go/bigquery v1.4.0/go.mod h1:S8dzgnTigyfTmLBfrtrhyYhwRxG72rYxvftPBK2Dvzc=
cloud.google.com/go/datastore v1.1.0/go.mod h1:umbIZjpQpHh4hmRpGhH4tLFup+FVzqBi1b3c64qFpCk=
cloud.google.com/go/firestore v1.1.0/go.mod h1:ulACoGHTpvq5r8rxGJ4ddJZBZqakUQqClKRT5SZwBmk=
cloud.google.com/go/pubsub v1.2.0/go.mod h1:jhfEVHT8odbXTkndysNHCcx0awwzvfOlguIAii9o8iA=
cloud.google.com/go/storage v1.6.0/go.mod h1:N7U0C8pVQ/+NIKOBQyamJIeKQKkZ+mxpohlUTyfDhBk=
dmitri.shuralyov.com/gpu/mtl v0.0.0-20190408044501-666a987793e9/go.mod h1:H6x//7gZCb22OMCxBHrMx7a5I7Hp++hsVxbQ4BYO7hU=
github.com/Azure/azure-sdk-for-go v43.0.0+incompatible h1:/wSNCu0e6EsHFR4Qa3vBEBbicaprEHMyyga9g8RTULI=
github.com/Azure/azure-sdk-for-go v43.0.0+incompatible/go.mod h1:9XXNKU+eRnpl9moKnB4QOLf1HestfXbmab5FXxiDBjc=
github.com/Azure/go-ansiterm v0.0.0-20170929234023-d6e3b3328b78 h1:w+iIsaOQNcT7OZ575w+acHgRric5iCyQh+xv+KJ4HB8=
github.com/Azure/go-ansiterm v0.0.0-20170929234023-d6e3b3328b78/go.mod h1:LmzpDX56iTiv29bbRTIsUNlaFfuhWRQBWjQdVyAevI8=
github.com/Azure/go-autorest v14.2.0+incompatible h1:V5VMDjClD3GiElqLWO7mz2MxNAK/vTfRHdAubSIPRgs=
github.com/Azure/go-autorest v14.2.0+incompatible/go.mod h1:r+4oMnoxhatjLLJ6zxSWATqVooLgysK6ZNox3g/xq24=
github.com/Azure/go-autorest/autorest v0.11.12 h1:gI8ytXbxMfI+IVbI9mP2JGCTXIuhHLgRlvQ9X4PsnHE=
github.com/Azure/go-autorest/autorest v0.11.12/go.mod h1:eipySxLmqSyC5s5k1CLupqet0PSENBEDP93LQ9a8QYw=
github.com/Azure/go-autorest/autorest/adal v0.9.5 h1:Y3bBUV4rTuxenJJs41HU3qmqsb+auo+a3Lz+PlJPpL0=
github.com/Azure/go-autorest/autorest/adal v0.9.5/go.mod h1:B7KF7jKIeC9Mct5spmyCB/A8CG/sEz1vwIRGv/bbw7A=
github.com/Azure/go-autorest/autorest/date v0.3.0 h1:7gUk1U5M/CQbp9WoqinNzJar+8KY+LPI6wiWrP/myHw=
github.com/Azure/go-autorest/autorest/date v0.3.0/go.mod h1:BI0uouVdmngYNUzGWeSYnokU+TrmwEsOqdt8Y6sso74=
github.com/Azure/go-autorest/autorest/mocks v0.4.1 h1:K0laFcLE6VLTOwNgSxaGbUcLPuGXlNkbVvq4cW4nIHk=
github.com/Azure/go-autorest/autorest/mocks v0.4.1/go.mod h1:LTp+uSrOhSkaKrUy935gNZuuIPPVsHlr9DSOxSayd+k=
github.com/Azure/go-autorest/autorest/to v0.2.0 h1:nQOZzFCudTh+TvquAtCRjM01VEYx85e9qbwt5ncW4L8=
github.com/Azure/go-autorest/autorest/to v0.2.0/go.mod h1:GunWKJp1AEqgMaGLV+iocmRAJWqST1wQYhyyjXJ3SJc=
github.com/Azure/go-autorest/autorest/validation v0.1.0 h1:ISSNzGUh+ZSzizJWOWzs8bwpXIePbGLW4z/AmUFGH5A=
github.com/Azure/go-autorest/autorest/validation v0.1.0/go.mod h1:Ha3z/SqBeaalWQvokg3NZAlQTalVMtOIAs1aGK7G6u8=
github.com/Azure/go-autorest/logger v0.2.0 h1:e4RVHVZKC5p6UANLJHkM4OfR1UKZPj8Wt8Pcx+3oqrE=
github.com/Azure/go-autorest/logger v0.2.0/go.mod h1:T9E3cAhj2VqvPOtCYAvby9aBXkZmbF5NWuPV8+WeEW8=
github.com/Azure/go-autorest/tracing v0.6.0 h1:TYi4+3m5t6K48TGI9AUdb+IzbnSxvnvUMfuitfgcfuo=
github.com/Azure/go-autorest/tracing v0.6.0/go.mod h1:+vhtPC754Xsa23ID7GlGsrdKBpUA79WCAKPPZVC2DeU=
```

go.sum考虑的是依赖防篡改的问题。第一次拉取包含go.sum的工程，项目运行之后通常会去拉取依赖包。此时会通过sha256（内容摘要算法）计算依赖包的hash值，并与go.sum记录值进行比较。go.sum期望行为是Append-Only配合工具生成。

### 命令行与工作流
go mod目前也是Go官方工具链中的一员，通过go help mod可以查看帮助信息：
```bash
$ go help mod
Go mod provides access to operations on modules.

Note that support for modules is built into all the go commands,
not just 'go mod'. For example, day-to-day adding, removing, upgrading,
and downgrading of dependencies should be done using 'go get'.
See 'go help modules' for an overview of module functionality.

Usage:

	go mod <command> [arguments]

The commands are:

	download    download modules to local cache
	edit        edit go.mod from tools or scripts
	graph       print module requirement graph
	init        initialize new module in current directory
	tidy        add missing and remove unused modules
	vendor      make vendored copy of dependencies
	verify      verify dependencies have expected content
	why         explain why packages or modules are needed

Use "go help mod <command>" for more information about a command.
```

并且mod的概念也融入之前的Go官方工具，例如go get、go list、go build、go test等。当运行这些相关命令的时候，会配合Go Modules的算法和行为。

go mod的常用工作流大致如下：
1. 创建新的module：`go mod init`首次接入go mod的项目可以通过init命令完成，init命令会解析项目源码中go文件的import信息，并根据MVS相关算法，计算出相关依赖并最终生成go.mod和go.sum。
2. 列出当前module下的所有依赖：`go list -m -json all`
3. 升级/降级一个依赖到指定版本：`go get -u github.com/pkg/errors@v0.9.0`
4. 升级所有依赖：`go get -u all`
5. 下载依赖包：`go mod download`
6. 下载依赖包并且精简mod和sum的内容：`go mod tidy`

### 代理
Go Modules允许企业自建代理服务器，用于存放依赖包，减小网络开销。

#### GOPROXY
```bash
GOPROXY="https://proxy.golang.org,direct"
```

GOPROXY可以提供一个服务列表，表示Go依赖的中央镜像仓库地址。在运行Go相关命令行时候，如果需要拉取一个依赖，会同时从服务列表的地址中选取合适的依赖下载。GOPROXY用于解决因为网络原因拉取依赖过慢的问题。direct表示根据导入路径直接拉取，不走代理。

#### GOSUMDB
```bash
GOSUMDB="sum.golang.google.cn"
```

为了防止拉取到被篡改的依赖，Go提供了sum db服务来校验依赖的正确性。sum db基于内容摘要算法，计算出hash值用于校验。

#### GONOPROXY
```bash
GONOPROXY="*.private.org"
```

在Go1.13中首次引入，考虑到很多公司私有依赖的保密性原因。列表中的依赖名固定不走代理。

#### GONOSUMDB
```bash
GONOSUMDB="*.private.org"
```

在Go1.13中首次引入，考虑到很多公司私有依赖的保密性原因。列表中的依赖名固定不做校验。

#### GOPRIVATE
```bash
GOPRIVATE="*.private.org"
```

GOPRIVATE是GONOPROXY和GONOSUMDB的综合，主要用于简化配置。GOPRIVATE列表中的依赖名自动进入GONOPROXY和GONOSUMDB。

## 总结
本文详细介绍了Go依赖管理的历史演进，从时间线上依次概述五个阶段的重点突破和待解决的问题。同时也深入解析了Go Modules的设计核心原理：语义导入版本+最小版本选择。另一方面，本文也简要介绍了Go Modules的日常工作流、命令行和代理配置，在工作中将会非常有用！

## 参考
1. [Golang版本管理系列翻译11篇全](https://github.com/vikyd/note/tree/master/go_and_versioning)
2. [vgo](https://research.swtch.com/vgo)

## 脚注
[^1]: https://twitter.com/_rsc/status/1022588240501661696