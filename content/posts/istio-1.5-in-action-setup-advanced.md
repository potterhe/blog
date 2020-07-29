---
title: "Istio：ingressGateway"
date: 2020-05-25T14:40:50+08:00
draft: false
tags: ["istio", "实战", "ingressGateway"]
slug: istio-1.5-in-action-setup-advanced
---

## 环境

- istio 1.5.2

## 多ingressGateway

什么场景会需要多个 ingressGateway

- 多租户。[Istio 租户模型- Namespace tenancy](https://istio.io/docs/ops/deployment/deployment-models/#namespace-tenancy) 描述了“命名空间”租户模型，不同集群的相同 namespace 视为一个租户。这种模型下，各租户的资源应当是隔离的，ingressgateway和各种secret（如证书）就是租户的资源。
- 同一个租户，需要公共/私有的入口。私有ingressGateway 在公有云上可以理解为一个内网的lb，当有业务不在k8s集群内时，集群外的业务要访问集群内的业务，一般是通过这个内网的lb；多个云SP通过专线互联时，同一个云跨AZ时；这些都对应多集群网格的构建场景，私有的ingressGateway是非常必要的，特别是Gateway方式的多集群网格，更应该选择内网lb作为集群间互通的入口，而不是mTLS的公网gateway。

### components.ingressGateways

好消息是：`components.ingressGateways` 是一个list, 可见设计意图里包含多个 ingreessgateway。相应的，`components.egressGateways` 也是同样的。

```sh
$ istioctl profile dump --config-path components.ingressGateways
- enabled: true
  k8s:
    hpaSpec:
      maxReplicas: 5
      metrics:
      - resource:
          name: cpu
          targetAverageUtilization: 80
        type: Resource
      minReplicas: 1
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: istio-ingressgateway
    resources:
      limits:
        cpu: 2000m
        memory: 1024Mi
      requests:
        cpu: 100m
        memory: 128Mi
    strategy:
      rollingUpdate:
        maxSurge: 100%
        maxUnavailable: 25%
  name: istio-ingressgateway
```

坏消息是：上面定义的多个ingressgateway 的 label 不生效(截止1.5.2)，这导致同一 namespace的多个ingressgateway无实际意义。关于此的讨论请见以下列举的issue和pull request

- https://github.com/istio/istio/issues/23341
- https://github.com/istio/istio/issues/23392
- https://github.com/istio/istio/pull/22981

归纳

- @linsun 回复说：“目前(1.5.2)，我们不支持在同一名称空间中安装多个网关-也不是一个好的实践。 Ingress secret 和访问权限应与控制平面分开。”，也就是说一个namespace只支持一个ingressgateway，可以通过这个方式曲折的实现公网/内网的需求。
- 1.6 会支持

### values.gateways.istio-ingressgateway

通过分析istioctl manifest 的源码，istioctl在生成manifest时，对`components.ingressGateways`列表里定义的ingressgateway会拆分成一个个独立的组件,分别用 install/kubernetes/operator/charts/gateways/istio-ingress/ 这个charts进行渲染。渲染之前会进行一系统helm templates 相关的values覆盖逻辑。

`values.gateways.istio-ingressgateway` 不是list 的设计，只是一个ingressgateway，对应于charts:install/kubernetes/operator/charts/gateways/istio-ingress。可以理解为它是ingressgate的模板。如果`components.ingressGateways` 里没有定义，就不会生成相关manifest。

`values.gateways.istio-ingressgateway` 会作用于`components.ingressGateways` 定义的每一个网关。所以如果某些定义是所有网关通过的，应该放置于这里。

## istio 1.6

istio-1.6.0 于2020年5月22日发布了，对安装中的很多地方做了修订，这就到此为止。
