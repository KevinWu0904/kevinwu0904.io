---
title: Helm——Kubernetes的包管家
toc: true
authors:
  - kevinwu
tags:
  - k8s
  - helm
series:
  - 云原生
date: '2021-06-21'
lastmod: '2021-06-21'
draft: false
---

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-k8s-helm/20210618154524.png)

如果将[Kubernetes](https://kubernetes.io/)比作是云原生时代的**云操作系统**，那么[Helm](https://helm.sh/)就是这个云操作系统的**包管家**。Helm之于Kubernetes，就如同apt之于ubuntu，yum之于centOS。

## 为什么需要Helm
Kubernetes将**编排**这一颇为抽象的概念，通过其设计的[资源对象](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)具象化了。笔者在[《Kubernetes架构设计与核心原理》](../k8s-architecture)一文中阐述了Kubernetes的声明式API和控制器模式是驱动其源源不断的编排动力，而所有的控制器都会针对一种或者多种资源对象进行调协。由此，我们可以看出资源对象在整个Kubernetes中处于核心地位。

Kubernetes通过[yaml](https://en.wikipedia.org/wiki/YAML)文件来描述资源对象。以官方示例的nginx-deployment对象为例：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
如上，Kubernetes的yaml文件会定义一个或者多个符合Kubernetes Spec设计的资源对象，而每个资源对象在Kubernetes系统中则描述了一种特定的编排能力。编辑完yaml之后，开发者可以通过kubectl来应用它，例如：
```bash
$ kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

yaml文件所表述的资源对象会进入Kubernetes的系统中，成功创建之后会输出如下类似信息：
```bash
deployment.apps/nginx-deployment created
```

综上所述，Kubernetes通过yaml管理资源对象，每个资源对象都由一段yaml文件内容表示。然而，**一个完整的、生产环境可用的Kubernetes App在Kubernetes中通常包括数个yaml资源对象**，例如：[MySQL Helm Chart](https://github.com/helm/charts/tree/master/stable/mysql/templates)

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-k8s-helm/20210620105821.png)

如图所示，在MySQL Helm Chart中，templates定义的yaml资源对象不仅有核心的Deployment，还包括如：PVC、ServiceAccount、Secrets、ConfigMap等等。

由此我们知道，Kubernetes之所以需要Helm，是因为Kubernetes只面向Kubernetes资源对象，而**Helm则真正意义上定义了Kuberentes App的边界**，Helm通过Chart完整描述了运行在Kubernetes之上的App所需要的所有内容，并个性化的提供**values.yaml**作为面向用户的配置文件，从而完整安装了Kubernetes App。

Helm简化了Kubernetes繁琐的yaml管理，极大降低了开发者的运维成本！

## Helm的三大核心概念
Helm的三大核心概念分别是：**Chart**、**Repository**、**Release**。

* **Chart**代表着Helm包。它包含在Kubernetes集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是Homebrew formula，Apt dpkg，或Yum RPM在Kubernetes中的等价物。
* **Repository**是用来存放和共享Charts的地方。它就像Perl的CPAN档案库网络或是Fedora的软件包仓库，只不过它是供Kubernetes包所使用的。
* **Release**是运行在Kubernetes集群中的Chart的实例。一个Chart通常可以在同一个集群中安装多次。每一次安装都会创建一个新的Release。以MySQL Chart为例，如果你想在你的集群中运行两个数据库，你可以安装该Chart两次。每一个数据库都会拥有它自己的Release 和Release Name。

在了解了上述这些概念以后，我们就可以这样来解释Helm：

Helm安装Charts到Kubernetes集群中，每次安装都会创建一个新的Release。你可以在Helm的Chart Repositories中寻找新的Chart。

## Helm v2和Helm v3
[Helm v3](https://helm.sh/blog/helm-3-released/)是目前Helm最新的大版本，于2019年11月正式Release稳定版本。Helm v3是在Helm v2的基础上，随着Kubernetes自身的发展而提出的重大升级，**与Helm v2不兼容**。不过为了减少升级带来的影响，Helm官方发布了Helm Chart v2 to v3的[插件](https://github.com/helm/helm-2to3)。

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-k8s-helm/20210620140930.png)

Helm v3中最重要的改动是：**[废除Tiller组件](https://v3.helm.sh/docs/faq/changes_since_helm2/)**。Tiller在Helm v2版本中的主要作用有：

1. 监听Helm客户端的请求。
2. 将Chart和Configuration合并，构建Release。
3. 将Chart安装到Kubernetes集群，并跟踪后续的Release。
4. 通过与Kubernetes交互升级/卸载Chart。

Tiller是一个**gRPC Server**，运行在Kubernetes集群内部。Tiller在不同团队共享的Kubernetes集群场景下扮演者重要作用，它允许不同的Operator操作相同的Release。

然而**随着Kubernetes v1.6版本默认打开RBAC（Role-Based Access Control）**，Tiller的管理变得更加困难。Helm官方为了保持Helm的简单性（例如:让初次使用Helm的用户无需考虑安全相关的因素）新增了一些默认权限配置。不幸的是，这使得Helm的用户可能被赋予了一些本不该有的权限。导致DevOps和SRE们需要更多学习额外的运维知识。

Helm官方吸取了来自社区成员的反馈和建议，他们发现Helm实际上无需维护这个In-Cluster的Operator，也就是Tiller组件。**Helm客户端可以直接与Kubernetes API Server交互获取信息，提交Helm Chart客户端，并且在Kubernetes中存储安装信息**。

移除Tiller组件之后，Helm的安全模型变得异常简单。Helm的权限将会复用Kubernetes的kube config文件中配置的权限，集群管理员也可以更加严格地控制用户的权限了。

![Helm v3 Architecture](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-k8s-helm/20210620205851.png)

## Helm v3的安装
Helm的安装方式主要有：

1. 二进制下载。
2. 脚本安装。
3. 包管理器安装。（不推荐）

Helm官方并不推荐使用包管理器安装，因为它们并不是由Helm官方项目所维护的内容。下面介绍Helm推荐的两种安装方式。

### 二进制下载
每个Helm的Release版本都提供了各种操作系统的二进制版本，这些版本可以手动下载和安装。

1. 在Helm的[Release版本列表](https://github.com/helm/helm/releases)中选择合适的OS版本并下载。
2. 解压。（以Linux为例：tar -zxvf helm-v3.0.0-linux-amd64.tar.gz）
3. 在解压的目录中找到helm二进制可执行文件，并移动到对应目录下。（mv linux-amd64/helm /usr/local/bin/helm）

### 脚本安装
Helm现在有个安装脚本可以自动拉取最新的Helm版本并在本地安装。(建议先把脚本下载到本地，查看脚本运行内容，判断无风险之后运行。)

```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

## Helm CLI工作流
### 搜索Chart
Helm允许从两种来源搜索目标Chart：

1. `helm search hub`：从CNCF的[Artifact Hub](https://artifacthub.io/)上面查找并罗列Helm Charts。
```bash
$ helm search hub wordpress
URL                                                 CHART VERSION APP VERSION DESCRIPTION
https://hub.helm.sh/charts/bitnami/wordpress        7.6.7         5.2.4       Web publishing platform for building blogs and ...
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.6.3        v0.6.3      Presslabs WordPress Operator Helm Chart
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.7.1        v0.7.1      A Helm chart for deploying a WordPress site on ...
```

2. `helm search repo`：从本地的（`helm repo list`）仓库中查找并罗列Helm Charts。
```bash
$ helm repo add brigade https://brigadecore.github.io/charts
"brigade" has been added to your repositories
$ helm search repo brigade
NAME                          CHART VERSION APP VERSION DESCRIPTION
brigade/brigade               1.3.2         v1.2.1      Brigade provides event-driven scripting of Kube...
brigade/brigade-github-app    0.4.1         v0.2.1      The Brigade GitHub App, an advanced gateway for...
brigade/brigade-github-oauth  0.2.0         v0.20.0     The legacy OAuth GitHub Gateway for Brigade
brigade/brigade-k8s-gateway   0.1.0                     A Helm chart for Kubernetes
brigade/brigade-project       1.0.0         v1.0.0      Create a Brigade project
brigade/kashti                0.4.0         v0.4.0      A Helm chart for Kubernetes
```

### 安装Chart
Helm通过`helm install`命令来安装Chart，最简单的方式只需要传入两个参数：你命名的Release名称和你希望安装的Chart名称。

```bash
$ helm install happy-panda bitnami/wordpress
NAME: happy-panda
LAST DEPLOYED: Tue Jan 26 10:27:17 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    happy-panda-wordpress.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w happy-panda-wordpress'

   export SERVICE_IP=$(kubectl get svc --namespace default happy-panda-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo "WordPress URL: http://$SERVICE_IP/"
   echo "WordPress Admin URL: http://$SERVICE_IP/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: user
  echo Password: $(kubectl get secret --namespace default happy-panda-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

### 查看Chart状态
Helm CLI不会等到所有资源对象都进入Ready状态之后才退出，我们可以通过`helm status`查看Release的状态。

```bash
$ helm status happy-panda
NAME: happy-panda
LAST DEPLOYED: Tue Jan 26 10:27:17 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    happy-panda-wordpress.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w happy-panda-wordpress'

   export SERVICE_IP=$(kubectl get svc --namespace default happy-panda-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo "WordPress URL: http://$SERVICE_IP/"
   echo "WordPress Admin URL: http://$SERVICE_IP/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: user
  echo Password: $(kubectl get secret --namespace default happy-panda-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

### 升级/回滚Chart
Helm通过`helm upgrade`命令来升级Chart。

```bash
$ helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```

每次的升级操作对Helm Chart的Release会产生新的Revision，Revision可以通过`helm history`查看：
```bash
$ helm install happy-panda

REVISION	UPDATED                 	STATUS    	CHART        	APP VERSION	DESCRIPTION
1       	Fri Jun 18 16:17:51 2021	superseded	happy-panda	  5.2.4      	Install complete
2       	Fri Jun 18 17:07:21 2021	superseded	happy-panda	  5.2.4      	Upgrade complete
3       	Fri Jun 18 17:33:57 2021	deployed  	happy-panda	  5.2.4      	Upgrade complete
```
由于Kubernetes的Chart可能会很大而且很复杂，因此Helm的升级操作仅会应用两次比较不同的部分，即**增量升级**。这一点从Revision的History中也能够看出。

当Helm升级之后遇到不符合预期的事情，也可以通过`helm rollback`命令快速回滚到之前的某个Revision。

```bash
$ helm rollback happy-panda 1
```

### 卸载Chart
Helm通过`helm uninstall`命令卸载一个已经安装的Chart Release。

```bash
$ helm uninstall happy-panda
```

在卸载之前，我们可以通过`helm list`查看当前Kubernetes集群中已经存在的Chart Release。
```bash
$ helm list
NAME            VERSION UPDATED                         STATUS          CHART
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
```

### 自定义Chart
创建Chart是一个App进行云原生化的重要手段，Helm通过`helm create`命令快速创建一个Chart模板供开发者去编辑，编辑完成之后的Chart即可通过`helm install`命令安装进Kubernetes集群。

```bash
$ helm create deis-workflow
Creating deis-workflow
```

另外值得一提的是，在编辑Chart的过程中，我们可以通过`helm lint`命令来查看当前Chart的语法格式是否正确。

当准备将Chart打包分发的时候，我们可以通过`helm package`命令来完成，紧接着通过`helm install`命令进行安装。
```bash
$ helm package deis-workflow
deis-workflow-0.1.0.tgz

$ helm install deis-workflow ./deis-workflow-0.1.0.tgz
```

## Helm Chart组成
Chart的语法设计基于[Go语言的模板](https://pkg.go.dev/text/template)，同时Helm也在此基础上进行了一定的扩展。

### Helm Chart目录结构综述
一个完整的Helm Chart主要包括如下内容：
```
wordpress/
  Chart.yaml          # 包含了chart信息的YAML文件
  LICENSE             # 可选: 包含chart许可证的纯文本文件
  README.md           # 可选: 可读的README文件
  values.yaml         # chart 默认的配置值
  values.schema.json  # 可选: 一个使用JSON结构的values.yaml文件
  charts/             # 包含chart依赖的其他chart
  crds/               # 自定义资源的定义
  templates/          # 模板目录， 当和values 结合时，可生成有效的Kubernetes manifest文件
  templates/NOTES.txt # 可选: 包含简要使用说明的纯文本文件
```

### Chart.yaml
`Chart.yaml`文件是Chart必需的，包括如下内容：
``` yaml
apiVersion: chart API 版本 （必需）
name: chart名称 （必需）
version: 语义化2 版本（必需）
kubeVersion: 兼容Kubernetes版本的语义化版本（可选）
description: 一句话对这个项目的描述（可选）
type: chart类型 （可选）
keywords:
  - 关于项目的一组关键字（可选）
home: 项目home页面的URL （可选）
sources:
  - 项目源码的URL列表（可选）
dependencies: # chart 必要条件列表 （可选）
  - name: chart名称 (nginx)
    version: chart版本 ("1.2.3")
    repository: （可选）仓库URL ("https://example.com/charts") 或别名 ("@repo-name")
    condition: （可选） 解析为布尔值的yaml路径，用于启用/禁用chart (e.g. subchart1.enabled )
    tags: # （可选）
      - 用于一次启用/禁用 一组chart的tag
    import-values: # （可选）
      - ImportValue 保存源值到导入父键的映射。每项可以是字符串或者一对子/父列表项
    alias: （可选） chart中使用的别名。当你要多次添加相同的chart时会很有用
maintainers: # （可选）
  - name: 维护者名字 （每个维护者都需要）
    email: 维护者邮箱 （每个维护者可选）
    url: 维护者URL （每个维护者可选）
icon: 用做icon的SVG或PNG图片URL （可选）
appVersion: 包含的应用版本（可选）。不需要是语义化，建议使用引号
deprecated: 不被推荐的chart （可选，布尔值）
annotations:
  example: 按名称输入的批注列表 （可选）.
```

核心字段简要说明：

* apiVersion：对于至少需要Helm v3的Chart，apiVersion至少是v2。之前的版本是v1。
* appVersion：仅供参考，指代应用本身的版本。
* kubeVersion：支持的Kubernetes版本，可以进行一定的约束。例如：`>= 1.13.0 < 1.14.0 || >= 1.14.1 < 1.15.0`
* version：Chart自身的版本。
* deprecated：可用于描述该Chart是否已经被弃用。
* type：`application`或者`library`，库类型的Chart提供针对Chart构建的实用功能，不能被安装，通常也不包括任何资源对象。
* dependencies：Chart允许依赖其它Chart，这些依赖也可以放置到`charts/`目录下进行手动配置。

### templates/ 
`templates/`目录是定义Chart资源对象模板的核心目录，遵循Go语言的模板引擎规则。例如：
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

### values.yaml
`values.yaml`提供给用户自定义Chart配置的能力。例如：
```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

### values.schema.json
`values.schema.json`将会通过JSON格式的内容对Helm CLI（例如：`helm install`、`helm upgrade`、`helm lint`、`helm template`）进行schema校验。例如：
```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "image": {
      "description": "Container Image",
      "properties": {
        "repo": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      },
      "type": "object"
    },
    "name": {
      "description": "Service name",
      "type": "string"
    },
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": [
    "protocol",
    "port"
  ],
  "title": "Values",
  "type": "object"
}
```

### crds/
Kubernetes支持自定义资源对象[CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)。Helm v3中，CRD被视为特殊的资源对象，将会在所有的Chart安装之前安装。

注意：**CRD文件无法被模板化，必须是普通的yaml文件**!

```
crontabs/
  Chart.yaml
  crds/
    crontab.yaml
  templates/
    mycrontab.yaml
```

```yaml
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
```

由于CRD在Kubernetes中非常特殊，因此Helm对CRD采取极为保守的态度：

1. CRD从不重新安装。如果Helm确定`crds/`目录中的CRD已经存在，那么Helm不会安装或者升级。
2. CRD从不会在升级或回滚时安装。Helm只会在安装时创建CRD。
3. CRD从不会被删除。自动删除CRD会删除集群中所有命名空间中的所有CRD内容。因此Helm不会删除CRD。

## 参考
1. [Helm官方文档](https://helm.sh/zh/docs/)

## 总结
本文主要阐述了Helm在Kubernetes中的重要作用，并且简要概述了Helm的CLI工作流和Chart的组成。

另外值得一提的是，笔者在寻找参考资料的时候发现其实最好的参考资料就是[Helm官方文档](https://helm.sh/zh/docs/)，强烈建议读者们仔细翻阅！

## 后记
在写本篇博客的同时，笔者主要是参考的是：[Helm官方文档](https://helm.sh/zh/docs/)。并且意外发现官方中文版文档中一小处翻译错误，笔者随即提交了一个[PR](https://github.com/helm/helm-www/pull/1140)来修复这一小处翻译错误。虽然这看似是一个微不足道的小fix，但笔者同样能够感受到给社区贡献价值的喜悦！

![](https://kevinwu0904-blog-images.oss-cn-shanghai.aliyuncs.com/blogs-k8s-helm/20210625120426.png)

开源社区就是这样一个能够连接全世界各地开发者的优秀社区，笔者也希望自己能够在这条道路上面逐步前行，共勉！