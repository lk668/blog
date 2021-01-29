---
title: kubernetes中pod的自动伸缩
date: 2020-04-06
tags:
    - kubernetes
---

本文简单介绍一下kubernetes中pod的自动伸缩。
<!-- more -->


## 1. 简介

autoscaling/v1版本：Pod 水平的自动伸缩是基于CPU利用率自动伸缩deployment, replica set和replication controller中的pod数量进行。

autoscaling/v2beta2版本：除了CPU利用率这个指标，有很多custom metric https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md 也支持设置多个metric


## 2.工作机制

Pod 水平自动伸缩的实现是一个控制循环，由 controller manager 的 --horizontal-pod-autoscaler-sync-period 参数 指定周期（默认值为15秒）。

每个周期内，controller manager 根据每个 HorizontalPodAutoscaler 定义中指定的指标查询资源利用率。 controller manager 可以从 resource metrics API（每个pod 资源指标）和 custom metrics API（其他指标）获取指标。
> 对于每个 pod 的资源指标（如 CPU），控制器从资源指标 API 中获取每一个 HorizontalPodAutoscaler 指定 的 pod 的指标，然后，如果设置了目标使用率，控制器获取每个 pod 中的容器资源使用情况，并计算资源使用率。 如果使用原始值，将直接使用原始数据（不再计算百分比）。 然后，控制器根据平均的资源使用率或原始值计算出缩放的比例，进而计算出目标副本数

## 3.算法细节
从最基本的角度来看，pod 水平自动缩放控制器跟据当前指标和期望指标来计算缩放比例。

期望副本数 = ceil[当前副本数 * ( 当前指标 / 期望指标 )] （向上取整）

```
例如，当前指标为200m，目标设定值为100m,那么由于200.0 / 100.0 == 2.0， 副本数量将会翻倍。 如果当前指标为50m，副本数量将会减半，因为50.0 / 100.0 == 0.5。
``` 

1.如果计算出的缩放比例接近1.0（跟据--horizontal-pod-autoscaler-tolerance 参数全局配置的容忍值，默认为0.1）， 将会放弃本次缩放

2.如果有未就绪的pod（比如说在init）或者有任何pod的指标缺失。
- 1.我们首先不考虑未就绪和缺少指标的的pod，计算出缩放的方案   
- 2.再次计算缩放方案，将尚未就绪的pods默认消耗了指标的0%。对指标缺失的pod，在需要缩小时假设这些 pod 消耗了目标值的 100%， 在需要放大时假设这些 pod 消耗了0%目标值。然后重新计算缩放比例。 如果新的比率与缩放方向相反，或者在容忍范围内，则跳过缩放。 否则，我们使用新的缩放比例。

3.如果创建 HorizontalPodAutoscaler 时指定了多个指标， 那么会按照每个指标分别计算缩放副本数，取最大的进行缩放。 如果任何一个指标无法顺利的计算出缩放副本数（比如，通过 API 获取指标时出错）， 那么本次缩放会被跳过。

## 4. rolling upgrade

Currently in Kubernetes, it is possible to perform a rolling update by using the deployment object. Can't perform a rolling update on ReplicationController.

这里需要注意的是，如果Deployment设置为2，你的hpa的--min=1，那么如果hpa检测到负载不高的时候，会自动缩放到1。

## 5. Downscale Delay
--horizontal-pod-autoscaler-downscale-stabilization: The value for this option is a duration that specifies how long the autoscaler has to wait before another downscale operation can be performed after the current one has completed. The default value is 5 minutes (5m0s).

## 6. API resource/custom/external metrics
https://github.com/kubernetes/metrics/blob/master/IMPLEMENTATIONS.md#custom-metrics-api

### 6.1 Resource Metrics API

- Metrics Server: a lighter-weight in-memory server specifically for the resource metrics API.

### 6.2 Custom/External Metrics API

NB: None of the below implementations are officially part of Kubernetes. They are listed here for convenience.

- Prometheus Adapter. An implementation of the custom metrics API that attempts to support arbitrary metrics following a set label and naming scheme.

- Microsoft Azure Adapter. An implementation of the custom metrics API that allows you to retrieve arbitrary metrics from Azure Monitor.


- Datadog Cluster Agent. Implementation of the external metrics provider, using Datadog as a backend for the metrics. Coming soon: Implementation of the custom metrics provider to support in-cluster metrics collected by the Datadog Agents.

- Kube Metrics Adapter. A general purpose metrics adapter for Kubernetes that can collect and serve custom and external metrics for Horizontal Pod Autoscaling. Provides the ability to scrape pods directly or from Prometheus through user defined queries. Also capable of serving external metrics from a number of sources including AWS' SQS and ZMON monitoring.


**Examples for custom metrics:**

Custom metrics has two type. Pods and Object

**1. Pods metrics**

It only supports target type of **AverageValue**.
```yaml
type: Pods
pods:
  metric:
    name: packets-per-second
  target:
    type: AverageValue
    averageValue: 1k
```

**2. Object metrics**

These metrics describe a different object in the same namespace, instead of describing pods. Object metrics support **target** types of both **Value** and **AverageValue.** With Value, the target is compared directly to the returned metric from the API. With AverageValue, the value returned from the custom metrics API is divided by the number of pods before being compared to the target. 

```yaml
type: Object
object:
  metric:
    name: requests-per-second
  describedObject:
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    name: main-route
  target:
    type: Value
    value: 2k
```

## 7. 实践应用-（metrics-server）

为hpa提供CPU/MEM数据

```bash

1. Install via kubectl command
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

2. Install via helm
https://github.com/helm/charts/tree/master/stable/metrics-server

```

## 8. 实践应用-（prometheus)

### 8.1. Install prometheus
```
1. install prometheus
2. install promethus-adaptor
```

### 8.2 Usage

1. Get all custom metrics
```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"
```

2. Get specific metric detail
```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/cpu_usage"|jq
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/cpu_usage"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "load-generator-5fb4fb465b-r959b",
        "apiVersion": "/v1"
      },
      "metricName": "cpu_usage",
      "timestamp": "2020-06-02T12:33:43Z",
      "value": "0"
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "php-apache-5bfb859df7-9cvsq",
        "apiVersion": "/v1"
      },
      "metricName": "cpu_usage",
      "timestamp": "2020-06-02T12:33:43Z",
      "value": "0"
    }
  ]
}
capv@worker-ndc-2-control-plane-bbqk7 [ ~ ]$
```