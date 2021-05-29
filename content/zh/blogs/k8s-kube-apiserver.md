---
title: 深入理解Kubernetes API Server
toc: true
authors:
  - kevinwu
tags:
  - k8s
date: '2021-04-11'
lastmod: '2021-04-11'
draft: false
---

![](/images/blogs-k8s-common/k8s-logo.png)

kube-apiserver是整个Kubernetes架构中的核心组件，也是唯一与存储组件etcd交互的服务。kube-apiserver负责集群中Kubernetes资源对象的增删改查，并对外提供资源变更的事件通知，驱动集群中形形色色的控制器，最终得以完成集群编排。

注：本文内容基于Kubernetes v1.20.5版本。

## Endpoint概览
kube-apiserver默认暴露6443端口，笔者通过[Postman](https://www.postman.com/)访问该端口，得到kube-apiserver的Endpoint概览如下：
```json
GET https://<master ip>:6443/ # 注：需要获取cluster-admin Role的service account token，作为请求的Authorization
{
    "paths": [
        "/.well-known/openid-configuration",
        "/api",
        "/api/v1",
        "/apis",
        "/apis/",
        "/apis/admissionregistration.k8s.io",
        "/apis/admissionregistration.k8s.io/v1",
        "/apis/admissionregistration.k8s.io/v1beta1",
        "/apis/apiextensions.k8s.io",
        "/apis/apiextensions.k8s.io/v1",
        "/apis/apiextensions.k8s.io/v1beta1",
        "/apis/apiregistration.k8s.io",
        "/apis/apiregistration.k8s.io/v1",
        "/apis/apiregistration.k8s.io/v1beta1",
        "/apis/apps",
        "/apis/apps/v1",
        "/apis/authentication.k8s.io",
        "/apis/authentication.k8s.io/v1",
        "/apis/authentication.k8s.io/v1beta1",
        "/apis/authorization.k8s.io",
        "/apis/authorization.k8s.io/v1",
        "/apis/authorization.k8s.io/v1beta1",
        "/apis/autoscaling",
        "/apis/autoscaling/v1",
        "/apis/autoscaling/v2beta1",
        "/apis/autoscaling/v2beta2",
        "/apis/batch",
        "/apis/batch/v1",
        "/apis/batch/v1beta1",
        "/apis/certificates.k8s.io",
        "/apis/certificates.k8s.io/v1",
        "/apis/certificates.k8s.io/v1beta1",
        "/apis/coordination.k8s.io",
        "/apis/coordination.k8s.io/v1",
        "/apis/coordination.k8s.io/v1beta1",
        "/apis/discovery.k8s.io",
        "/apis/discovery.k8s.io/v1beta1",
        "/apis/events.k8s.io",
        "/apis/events.k8s.io/v1",
        "/apis/events.k8s.io/v1beta1",
        "/apis/extensions",
        "/apis/extensions/v1beta1",
        "/apis/flowcontrol.apiserver.k8s.io",
        "/apis/flowcontrol.apiserver.k8s.io/v1beta1",
        "/apis/networking.k8s.io",
        "/apis/networking.k8s.io/v1",
        "/apis/networking.k8s.io/v1beta1",
        "/apis/node.k8s.io",
        "/apis/node.k8s.io/v1",
        "/apis/node.k8s.io/v1beta1",
        "/apis/policy",
        "/apis/policy/v1beta1",
        "/apis/rbac.authorization.k8s.io",
        "/apis/rbac.authorization.k8s.io/v1",
        "/apis/rbac.authorization.k8s.io/v1beta1",
        "/apis/scheduling.k8s.io",
        "/apis/scheduling.k8s.io/v1",
        "/apis/scheduling.k8s.io/v1beta1",
        "/apis/storage.k8s.io",
        "/apis/storage.k8s.io/v1",
        "/apis/storage.k8s.io/v1beta1",
        "/healthz",
        "/healthz/autoregister-completion",
        "/healthz/etcd",
        "/healthz/log",
        "/healthz/ping",
        "/healthz/poststarthook/aggregator-reload-proxy-client-cert",
        "/healthz/poststarthook/apiservice-openapi-controller",
        "/healthz/poststarthook/apiservice-registration-controller",
        "/healthz/poststarthook/apiservice-status-available-controller",
        "/healthz/poststarthook/bootstrap-controller",
        "/healthz/poststarthook/crd-informer-synced",
        "/healthz/poststarthook/generic-apiserver-start-informers",
        "/healthz/poststarthook/kube-apiserver-autoregistration",
        "/healthz/poststarthook/priority-and-fairness-config-consumer",
        "/healthz/poststarthook/priority-and-fairness-config-producer",
        "/healthz/poststarthook/priority-and-fairness-filter",
        "/healthz/poststarthook/rbac/bootstrap-roles",
        "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
        "/healthz/poststarthook/start-apiextensions-controllers",
        "/healthz/poststarthook/start-apiextensions-informers",
        "/healthz/poststarthook/start-cluster-authentication-info-controller",
        "/healthz/poststarthook/start-kube-aggregator-informers",
        "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
        "/livez",
        "/livez/autoregister-completion",
        "/livez/etcd",
        "/livez/log",
        "/livez/ping",
        "/livez/poststarthook/aggregator-reload-proxy-client-cert",
        "/livez/poststarthook/apiservice-openapi-controller",
        "/livez/poststarthook/apiservice-registration-controller",
        "/livez/poststarthook/apiservice-status-available-controller",
        "/livez/poststarthook/bootstrap-controller",
        "/livez/poststarthook/crd-informer-synced",
        "/livez/poststarthook/generic-apiserver-start-informers",
        "/livez/poststarthook/kube-apiserver-autoregistration",
        "/livez/poststarthook/priority-and-fairness-config-consumer",
        "/livez/poststarthook/priority-and-fairness-config-producer",
        "/livez/poststarthook/priority-and-fairness-filter",
        "/livez/poststarthook/rbac/bootstrap-roles",
        "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
        "/livez/poststarthook/start-apiextensions-controllers",
        "/livez/poststarthook/start-apiextensions-informers",
        "/livez/poststarthook/start-cluster-authentication-info-controller",
        "/livez/poststarthook/start-kube-aggregator-informers",
        "/livez/poststarthook/start-kube-apiserver-admission-initializer",
        "/logs",
        "/metrics",
        "/openapi/v2",
        "/openid/v1/jwks",
        "/readyz",
        "/readyz/autoregister-completion",
        "/readyz/etcd",
        "/readyz/informer-sync",
        "/readyz/log",
        "/readyz/ping",
        "/readyz/poststarthook/aggregator-reload-proxy-client-cert",
        "/readyz/poststarthook/apiservice-openapi-controller",
        "/readyz/poststarthook/apiservice-registration-controller",
        "/readyz/poststarthook/apiservice-status-available-controller",
        "/readyz/poststarthook/bootstrap-controller",
        "/readyz/poststarthook/crd-informer-synced",
        "/readyz/poststarthook/generic-apiserver-start-informers",
        "/readyz/poststarthook/kube-apiserver-autoregistration",
        "/readyz/poststarthook/priority-and-fairness-config-consumer",
        "/readyz/poststarthook/priority-and-fairness-config-producer",
        "/readyz/poststarthook/priority-and-fairness-filter",
        "/readyz/poststarthook/rbac/bootstrap-roles",
        "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
        "/readyz/poststarthook/start-apiextensions-controllers",
        "/readyz/poststarthook/start-apiextensions-informers",
        "/readyz/poststarthook/start-cluster-authentication-info-controller",
        "/readyz/poststarthook/start-kube-aggregator-informers",
        "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
        "/readyz/shutdown",
        "/version"
    ]
}
```

kube-apiserver的路由设计遵循层级结构：
![](/images/blogs-k8s-kube-apiserver/endpoint.png)

这里的层级结构将Kubernetes资源对象以“Group/Version/Resource”的方式组织。

kube-apiserver的主要Endpoint功能如下：
1. /api：Kubernetes核心资源对象，如：Namespace、Node、Pod、ConfigMap、Secret等等。
2. /apis：Kubernetes扩展资源对象，如：Job、CronJob、Deployment、StatefulSet、DaemonSet等等。
3. /healthz、/livez、/readyz：健康检查。其中healthz已被弃用（自Kubernetes v1.16起），应当使用更为明确的/livez和/readyz（存活和就绪）。
4. /logs：日志相关。
5. /metrics：监控相关。

## 资源对象的版本管理

### 版本定义
兼容性是每个API性质的Server绕不开的问题，对于Kubernetes这一云原生的核心项目来说尤其重要。Kubernetes通过引入资源的版本定义字段，来对每种资源对象进行管理。

以HorizontalPodAutoscaler为例，v1.20.5版本的Kubernetes既支持v2beta1，也支持v2beta2。

v2beta1 yaml：
```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: AverageUtilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      target:
        kind: Value
        value: 10k
```

v2beta2 yaml：
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 版本转换
这里有个常见的疑问：那么Kubernetes内存中会同时维护资源对象的不同版本吗？答案是：不会，Kubernetes会通过版本转换，将不同的版本对象统一成包括所有字段的内部对象。

下面笔者会展示相关源码：
1. 资源对象的定义会存在多个版本，以HorizontalPodAutoscaler为例：

kubernetes/vendor/k8s.io/api/autoscaling/v2beta1/types.go：
```go
// HorizontalPodAutoscaler is the configuration for a horizontal pod
// autoscaler, which automatically manages the replica count of any resource
// implementing the scale subresource based on the metrics specified.
type HorizontalPodAutoscaler struct {
	metav1.TypeMeta `json:",inline"`
	// metadata is the standard object metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// spec is the specification for the behaviour of the autoscaler.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status.
	// +optional
	Spec HorizontalPodAutoscalerSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// status is the current information about the autoscaler.
	// +optional
	Status HorizontalPodAutoscalerStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

// HorizontalPodAutoscalerSpec describes the desired functionality of the HorizontalPodAutoscaler.
type HorizontalPodAutoscalerSpec struct {
	// scaleTargetRef points to the target resource to scale, and is used to the pods for which metrics
	// should be collected, as well as to actually change the replica count.
	ScaleTargetRef CrossVersionObjectReference `json:"scaleTargetRef" protobuf:"bytes,1,opt,name=scaleTargetRef"`
	// minReplicas is the lower limit for the number of replicas to which the autoscaler
	// can scale down.  It defaults to 1 pod.  minReplicas is allowed to be 0 if the
	// alpha feature gate HPAScaleToZero is enabled and at least one Object or External
	// metric is configured.  Scaling is active as long as at least one metric value is
	// available.
	// +optional
	MinReplicas *int32 `json:"minReplicas,omitempty" protobuf:"varint,2,opt,name=minReplicas"`
	// maxReplicas is the upper limit for the number of replicas to which the autoscaler can scale up.
	// It cannot be less that minReplicas.
	MaxReplicas int32 `json:"maxReplicas" protobuf:"varint,3,opt,name=maxReplicas"`
	// metrics contains the specifications for which to use to calculate the
	// desired replica count (the maximum replica count across all metrics will
	// be used).  The desired replica count is calculated multiplying the
	// ratio between the target value and the current value by the current
	// number of pods.  Ergo, metrics used must decrease as the pod count is
	// increased, and vice-versa.  See the individual metric source types for
	// more information about how each type of metric must respond.
	// +optional
	Metrics []MetricSpec `json:"metrics,omitempty" protobuf:"bytes,4,rep,name=metrics"`
}
```

kubernetes/vendor/k8s.io/api/autoscaling/v2beta2/types.go：
```go
// HorizontalPodAutoscaler is the configuration for a horizontal pod
// autoscaler, which automatically manages the replica count of any resource
// implementing the scale subresource based on the metrics specified.
type HorizontalPodAutoscaler struct {
	metav1.TypeMeta `json:",inline"`
	// metadata is the standard object metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// spec is the specification for the behaviour of the autoscaler.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status.
	// +optional
	Spec HorizontalPodAutoscalerSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// status is the current information about the autoscaler.
	// +optional
	Status HorizontalPodAutoscalerStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

// HorizontalPodAutoscalerSpec describes the desired functionality of the HorizontalPodAutoscaler.
type HorizontalPodAutoscalerSpec struct {
	// scaleTargetRef points to the target resource to scale, and is used to the pods for which metrics
	// should be collected, as well as to actually change the replica count.
	ScaleTargetRef CrossVersionObjectReference `json:"scaleTargetRef" protobuf:"bytes,1,opt,name=scaleTargetRef"`
	// minReplicas is the lower limit for the number of replicas to which the autoscaler
	// can scale down.  It defaults to 1 pod.  minReplicas is allowed to be 0 if the
	// alpha feature gate HPAScaleToZero is enabled and at least one Object or External
	// metric is configured.  Scaling is active as long as at least one metric value is
	// available.
	// +optional
	MinReplicas *int32 `json:"minReplicas,omitempty" protobuf:"varint,2,opt,name=minReplicas"`
	// maxReplicas is the upper limit for the number of replicas to which the autoscaler can scale up.
	// It cannot be less that minReplicas.
	MaxReplicas int32 `json:"maxReplicas" protobuf:"varint,3,opt,name=maxReplicas"`
	// metrics contains the specifications for which to use to calculate the
	// desired replica count (the maximum replica count across all metrics will
	// be used).  The desired replica count is calculated multiplying the
	// ratio between the target value and the current value by the current
	// number of pods.  Ergo, metrics used must decrease as the pod count is
	// increased, and vice-versa.  See the individual metric source types for
	// more information about how each type of metric must respond.
	// If not set, the default metric will be set to 80% average CPU utilization.
	// +optional
	Metrics []MetricSpec `json:"metrics,omitempty" protobuf:"bytes,4,rep,name=metrics"`

	// behavior configures the scaling behavior of the target
	// in both Up and Down directions (scaleUp and scaleDown fields respectively).
	// If not set, the default HPAScalingRules for scale up and scale down are used.
	// +optional
	Behavior *HorizontalPodAutoscalerBehavior `json:"behavior,omitempty" protobuf:"bytes,5,opt,name=behavior"`
}
```

对比v2beta1和v2beta2的HorizontalPodAutoscaler定义，区别是v2beta2的HorizontalPodAutoscalerSpec里面新增了 `Behavior *HorizontalPodAutoscalerBehavior`

2. Kubernetes通过conversion-gen自动生成版本转换的代码，以HorizontalPodAutoscaler为例：

kubernetes/pkg/apis/autoscaling/v2beta1/zz_generated.conversion.go：
```go
func autoConvert_v2beta1_HorizontalPodAutoscaler_To_autoscaling_HorizontalPodAutoscaler(in *v2beta1.HorizontalPodAutoscaler, out *autoscaling.HorizontalPodAutoscaler, s conversion.Scope) error {
	out.ObjectMeta = in.ObjectMeta
	if err := Convert_v2beta1_HorizontalPodAutoscalerSpec_To_autoscaling_HorizontalPodAutoscalerSpec(&in.Spec, &out.Spec, s); err != nil {
		return err
	}
	if err := Convert_v2beta1_HorizontalPodAutoscalerStatus_To_autoscaling_HorizontalPodAutoscalerStatus(&in.Status, &out.Status, s); err != nil {
		return err
	}
	return nil
}
```

kubernetes/pkg/apis/autoscaling/v2beta2/zz_generated.conversion.go：
```go
func autoConvert_v2beta2_HorizontalPodAutoscaler_To_autoscaling_HorizontalPodAutoscaler(in *v2beta2.HorizontalPodAutoscaler, out *autoscaling.HorizontalPodAutoscaler, s conversion.Scope) error {
	out.ObjectMeta = in.ObjectMeta
	if err := Convert_v2beta2_HorizontalPodAutoscalerSpec_To_autoscaling_HorizontalPodAutoscalerSpec(&in.Spec, &out.Spec, s); err != nil {
		return err
	}
	if err := Convert_v2beta2_HorizontalPodAutoscalerStatus_To_autoscaling_HorizontalPodAutoscalerStatus(&in.Status, &out.Status, s); err != nil {
		return err
	}
	return nil
}
```

从源码可以看出，这两个自动生成的API会将HPA的不同版本转换成相同的内部HPA：
kubernetes/pkg/apis/autoscaling/types.go：
```go
// HorizontalPodAutoscaler is the configuration for a horizontal pod
// autoscaler, which automatically manages the replica count of any resource
// implementing the scale subresource based on the metrics specified.
type HorizontalPodAutoscaler struct {
	metav1.TypeMeta
	// Metadata is the standard object metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta

	// Spec is the specification for the behaviour of the autoscaler.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status.
	// +optional
	Spec HorizontalPodAutoscalerSpec

	// Status is the current information about the autoscaler.
	// +optional
	Status HorizontalPodAutoscalerStatus
}

// HorizontalPodAutoscalerSpec describes the desired functionality of the HorizontalPodAutoscaler.
type HorizontalPodAutoscalerSpec struct {
	// ScaleTargetRef points to the target resource to scale, and is used to the pods for which metrics
	// should be collected, as well as to actually change the replica count.
	ScaleTargetRef CrossVersionObjectReference
	// minReplicas is the lower limit for the number of replicas to which the autoscaler
	// can scale down.  It defaults to 1 pod.  minReplicas is allowed to be 0 if the
	// alpha feature gate HPAScaleToZero is enabled and at least one Object or External
	// metric is configured.  Scaling is active as long as at least one metric value is
	// available.
	// +optional
	MinReplicas *int32
	// MaxReplicas is the upper limit for the number of replicas to which the autoscaler can scale up.
	// It cannot be less that minReplicas.
	MaxReplicas int32
	// Metrics contains the specifications for which to use to calculate the
	// desired replica count (the maximum replica count across all metrics will
	// be used).  The desired replica count is calculated multiplying the
	// ratio between the target value and the current value by the current
	// number of pods.  Ergo, metrics used must decrease as the pod count is
	// increased, and vice-versa.  See the individual metric source types for
	// more information about how each type of metric must respond.
	// +optional
	Metrics []MetricSpec

	// behavior configures the scaling behavior of the target
	// in both Up and Down directions (scaleUp and scaleDown fields respectively).
	// If not set, the default HPAScalingRules for scale up and scale down are used.
	// +optional
	Behavior *HorizontalPodAutoscalerBehavior
}
```

### 小结
综上所述，Kubernetes通过conversion-gen生成的版本转换代码，将用户在yaml中定义的不同版本对象转换成包含所有字段的统一内部对象，从而无需在内存中同时维护资源对象的多个版本。值得注意的是，最终内部对象在持久化到etcd中，会再经过一次反向转换变成用户定义的yaml对象，最终存储。

![](/images/blogs-k8s-kube-apiserver/conversion.png)

## kube-apiserver

### 三个组件
kube-apiserver包含三个组件：
1. KubeAPIServer
2. APIExtensionsServer
3. AggregatorServer

### 启动流程

## POST资源的流程

## 总结

## 参考
1. https://www.bookstack.cn/read/source-code-reading-notes/kubernetes-kube_apiserver.md