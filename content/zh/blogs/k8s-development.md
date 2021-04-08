---
title: 从0到1搭建Kubernetes源码调试环境
toc: true
authors:
  - kevinwu
tags:
  - k8s
date: '2021-04-04'
lastmod: '2021-04-04'
draft: false
---

![](/images/blogs-k8s-common/k8s-logo.png)

[Kubernetes](https://kubernetes.io/)的源码纷繁复杂，为了能够深入学习其原理，笔者尝试从0到1搭建一套完备的源码调试环境。不同于[gdb](https://www.gnu.org/software/gdb/)的全命令行终端调试，本套环境完全基于图形化界面，体验堪称完美！

## 原理概述
就笔者的从业经验来看，绝大部分服务端开发者日常编码都基于Windows/Mac上的IDE（集成开发环境），编码完成后再通过SSH的方式连接到远端Linux开发机运行。这种做法的优点在于：
1. Windows/Mac图形化的编码过程，配合IDE各种优秀的插件，能够充分释放生产力。
2. Linux OS下运行，与生产环境OS一致，更能检验代码的正确性。

基于以上事实，本套环境的设计思路如下：
1. 在平时日常开发的Windows/Mac宿主机上面安装Linux虚拟机。
2. 在Linux虚拟机中通过kubeadm安装单机版Kubernetes。
3. 通过[GoLand](https://www.jetbrains.com/go/)和[dlv](https://github.com/go-delve/delve)远程调试Linux虚拟机中的Kubernetes进程。

![设计思路](/images/blogs-k8s-development/k8s-development.png)

## 成果展示
1. 两台虚拟机实例
![](/images/blogs-k8s-development/vm.png)

2. 一个Kubernetes集群
![](/images/blogs-k8s-development/vm-k8s-cluster.png)

3. 单步调试kube-apiserver
![](/images/blogs-k8s-development/debug-kube-apiserver.png)

## 宿主机前置依赖
1. 笔者的宿主机是Macbook Pro（16英寸版本），机器配置如下：
![](/images/blogs-k8s-development/mac.png)
2. 在宿主机上安装VMWare Fusion，版本如下：
![](/images/blogs-k8s-development/vmware.png)
3. 在宿主机上安装GoLand，版本如下：
![](/images/blogs-k8s-development/goland.png)

总的来说，本套环境对于宿主机的要求并不多。理论上同样支持Windows OS，也支持免费虚拟机软件[VirtualBox](https://www.virtualbox.org/)。

## 虚拟机环境搭建
接下来开启我们的环境搭建之旅！笔者将会详细记录本次环境搭建的关键步骤，尤其会显式标注依赖软件的版本。

### 步骤一：安装虚拟机
笔者使用的虚拟机是Debian 10，基于ISO文件：debian-10.7.0-amd64-netinst.iso。来源于：[https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/)（注：该链接会显示最新的版本，截止2021年4月4日，当前最新版本是debian-10.9.0-amd64-netinst.iso）

虚拟机的安装过程不是重点就此略过。

注意：笔者在安装完虚拟机之后，又安装了例如：oh-my-zsh、git、vim、curl等等工具。这些工具或多或少是下面安装docker和kubeadm的前置依赖，遇到报错之后再自行安装即可，主流程上面不会block。

### 步骤二：安装docker
docker是kubeadm的前置依赖，安装docker的步骤参考官方教程：[https://docs.docker.com/engine/install/debian/](https://docs.docker.com/engine/install/debian/)

1. 更新deb源，安装必要软件
```bash
$ apt-get update
$ apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y
```
![](/images/blogs-k8s-development/docker-install-1.png)

2. 添加docker官方GPG key
```bash
$ curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
![](/images/blogs-k8s-development/docker-install-2.png)

3. 添加docker amd64 deb源
```bash
$ echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```
![](/images/blogs-k8s-development/docker-install-3.png)

4.  更新deb源，查看docker-ce可选版本
```bash
$ apt-get update
$ apt-cache madison docker-ce
```
![](/images/blogs-k8s-development/docker-install-4.png)

5. 安装5:19.03.15~3-0~debian-buster版本
```bash
$ apt-get install docker-ce=5:19.03.15~3-0~debian-buster docker-ce-cli=5:19.03.15~3-0~debian-buster containerd.io
```
![](/images/blogs-k8s-development/docker-install-5.png)

6. 安装成功
```bash
$ systemctl status docker
```
![](/images/blogs-k8s-development/docker-install-6.png)

### 步骤三：安装kubeadm
[kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)是Kubernetes官方提供的安装工具，可用于快速部署Kubernetes集群。在本套源码调试环境中，我们需要一个基准Kubernetes集群，后续再基于该集群需要调试的CMD用Debug版本替换。

安装kubeadm的步骤参考官方教程：[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

1. 安装必要软件
```bash
$ apt-get install -y apt-transport-https ca-certificates curl
```
![](/images/blogs-k8s-development/kubeadm-install-1.png)

2. 添加Google Cloud GPG key
```bash
$ curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
![](/images/blogs-k8s-development/kubeadm-install-2.png)

3. 添加Kubernetes deb源
```bash
$ echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
```
![](/images/blogs-k8s-development/kubeadm-install-3.png)

4. 安装kubeadm、kubectl、kubelet
```bash
$ apt-get install kubeadm=1.20.5-00 kubectl=1.20.5-00 kubelet=1.20.5-00 -y
```
![](/images/blogs-k8s-development/kubeadm-install-4.png)

5. 安装成功
```bash
$ kubeadm
```
![](/images/blogs-k8s-development/kubeadm-install-5.png)

### 步骤四：安装Kubernetes集群
kubeadm只需要简单几个命令便能搭建生产可用的Kubernete集群。

1. 关闭swap
```bash
$ free -h
$ swapoff -a
```
![](/images/blogs-k8s-development/k8s-install-1.png)

2. 修改docker默认cgroup driver
```bash
$ vim /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

$ systemctl daemon-reload
$ systemctl restart docker
```

3. 重启虚拟机
```bash
$ reboot
```

4. 虚拟机Kubernetes-Control-Plane运行kubeadm init
```bash
$ kubeadm init
```
![](/images/blogs-k8s-development/k8s-install-2.png)

5. 虚拟机Kubernetes-Compute-Machine运行kubeadm join
```bash
$ kubeadm join ... # 在kubeadm init的末尾会输出kubeadm join的具体命令，复制粘贴即可
```
![](/images/blogs-k8s-development/k8s-install-3.png)

6. 虚拟机Kubernetes-Control-Plane安装[flannel](https://github.com/flannel-io/flannel) CNI
```bash
$ export KUBECONFIG=/etc/kubernetes/admin.conf

$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

7. flannel配置env
```bash
$ vim /run/flannel/subnet.env

# Control Plane
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.1.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true

# Compute Machine
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.2.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

8. 安装成功
```bash
$ kubectl get nodes
```
![](/images/blogs-k8s-development/vm-k8s-cluster.png)

### 步骤五：安装Go
1. wget安装包
```bash
$ wget https://dl.google.com/go/go1.16.3.linux-amd64.tar.gz
```

2. 解压并配置GOROOT、GOPATH和PATH
```bash
$ tar -zxvf go1.16.3.linux-amd64.tar.gz -C /usr/local/

$ export GOROOT="/usr/local/go"
$ export GOPATH="~/go"
$ export PATH="$PATH:$GOROOT/bin:$GOPATH/bin"
```

3. 安装成功
```bash
$ go env
```
![](/images/blogs-k8s-development/go-install-1.png)

### 步骤六：安装delve
1. 安装delve
```bash
$ go install github.com/go-delve/delve/cmd/dlv@latest
```

2. 安装成功
```bash
$ dlv
```

![](/images/blogs-k8s-development/delve-install-1.png)

### 小结
至此虚拟机环境搭建完成。在这一环节中，我们主要安装了一个基准Kubernetes集群，后续用于调试源码。

## 调试源码
接下来的篇幅以调试kube-apiserver组件为例进行说明，调试源码基于Kubernetes git v1.20.5版本。

1. 宿主机和虚拟机均拉取代码，切换分支到v1.20.5
```bash
$ mkdir -p $GOPATH/src/github.com/kubernetes
$ cd $GOPATH/src/github.com/kubernetes
$ git clone https://github.com/kubernetes/kubernetes.git
$ git check v1.20.5
```

2. 虚拟机编译包含DEBUG信息的CMD
```bash
$ apt install rsync -y # 需要安装rsync这个前置依赖

$ make all GOGCFLAGS="-N -l" GOLDFLAGS=""
```
参考Kubernetes该Issue：[https://github.com/kubernetes/kubernetes/issues/77527](https://github.com/kubernetes/kubernetes/issues/77527)
![](/images/blogs-k8s-development/debug-2.png)

3. 虚拟机查看kube-apiserver启动参数
```bash
$ cat /etc/kubernetes/manifests/kube-apiserver.yaml

apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 172.16.38.139:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.16.38.139
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
...
```

4. 虚拟机停止kube-apiserver的static pod
```bash
$ mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/
```
只需要把manifest目录下的配置文件移动到别的地方即可，kubelet会直接停止不在manifest目录下的static pod。

5. 通过dlv启动带DEBUG信息的kube-apiserver，不用担心，这里面参数都是复制上面yaml文件的内容
```bash
$ dlv exec --listen=:2345 --headless=true --api-version=2 _output/bin/kube-apiserver -- --advertise-address=172.16.38.139  --allow-privileged=true  --authorization-mode=Node,RBAC  --client-ca-file=/etc/kubernetes/pki/ca.crt  --enable-admission-plugins=NodeRestriction  --enable-bootstrap-token-auth=true  --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt  --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt  --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key  --etcd-servers=https://127.0.0.1:2379  --insecure-port=0  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt  --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt  --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key  --requestheader-allowed-names=front-proxy-client  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt  --requestheader-extra-headers-prefix=X-Remote-Extra-  --requestheader-group-headers=X-Remote-Group  --requestheader-username-headers=X-Remote-User  --secure-port=6443  --service-account-issuer=https://kubernetes.default.svc.cluster.local  --service-account-key-file=/etc/kubernetes/pki/sa.pub  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  --service-cluster-ip-range=10.96.0.0/12  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```
![](/images/blogs-k8s-development/debug-1.png)

6. 宿主机GoLand打开Kubernetes源码，配置go remote
![](/images/blogs-k8s-development/debug-3.png)

7. 启动！成功！撒花！
![](/images/blogs-k8s-development/debug-kube-apiserver.png)

## 总结
本文详细记录了笔者从0到1搭建Kubernetes源码调试环境的关键步骤，最终达到通过GUI调试的目的和体验，相信该教程对于Kubernetes开发者来说会有较大帮助。另一方面，笔者自己也希望通过这样一套调试环境加深自己对Kubernetes的理解，为完善自己的云原生技术栈而努力。

## 参考
1. [K8s远程调试，你的姿势对了吗？](https://cloud.tencent.com/developer/article/1624638)
2. [跟我学 K8S--代码: 调试 Kubernetes](https://segmentfault.com/a/1190000015807702)