---
title: "Istio：多集群安装"
date: 2020-05-27T14:40:50+08:00
draft: false
tags: ["istio", "1.6", "实战", "安装", "多集群"]
slug: istio-1.6-in-action-setup-multicluster
---

** 版本 1.8.0 官方文档[certificate management](https://istio.io/latest/docs/tasks/security/cert-management) 中的 “[Plug in CA Certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/)” 对此有了更好的描述。**

1. [部署模型](https://istio.io/docs/ops/deployment/deployment-models)
1. [Replicated control planes](https://istio.io/docs/setup/install/multicluster/gateways/)
1. [Shared control plane (single and multiple networks)](https://istio.io/docs/setup/install/multicluster/shared/)

## 环境

- istio 1.6.3
- k8s 1.16.3

## root-ca

为什么需要root-ca？多集群双向认证的必要条件，即使当前是单独部署，考虑到未来多集群的可能性，也应当做root-ca，intermediate-ca的处理。

官方文档有明确的提醒：**不可将“sample”的root-ca用于生产，需要构建私有的root-ca**。官方仓库有提供两个工具帮忙我们生成root-ca。

### 直接用 Makefile

[samples/certs/Makefile](https://github.com/istio/istio/blob/1.6.0/samples/certs/Makefile), 这个文件在master分支是没有的，在发布tag上有。然后执行以下命令, 会在当前目录生成4个文件

```sh
$ make root-ca
generating root-key.pem
Generating RSA private key, 4096 bit long modulus
..........................................................................++
...........................................................++
e is 65537 (0x10001)
generating root-cert.csr
generating root-cert.pem
Signature ok
subject=/O=Istio/CN=Root CA
Getting Private key

$ tree .
.
├── Makefile
├── root-ca.conf
├── root-cert.csr
├── root-cert.pem
└── root-key.pem
```

### samples/multicluster/setup-mesh.sh

[samples/multicluster/setup-mesh.sh](https://github.com/istio/istio/blob/1.6.0/samples/multicluster/setup-mesh.sh)。通过查看脚本源码，此脚本生成root-ca时，也是下载的Makefile 文件(TAG:1.6.0 下载的是 BRANCH:release-1.4 的文件，如果想用自己下载的Makefile，一定要放在WORKDIR/certs/ 目录下)，运行此脚本前，需先声明三个环境变量。

```sh
$ export WORKDIR=/PATH/TO/workdir
$ export MESH_ID=istio-1-6-0
$ export ORG_NAME=org-name
$ ./setup-mesh.sh prep-mesh
creating /PATH/TO/workdir/topology.yaml to describe the mesh topology
generating root certs for org-name
Generating RSA private key, 4096 bit long modulus
...............................................................++
.............................................................................................................................................++
e is 65537 (0x10001)
Signature ok
subject=/O=org-name/CN=Root CA
Getting Private key
Success!

    Add clusters to /PATH/TO/topology.yaml and run './setup-mesh.sh apply' to build the mesh

$ tree .
.
├── base.yaml
├── certs
│   ├── Makefile
│   ├── root-ca.conf
│   ├── root-cert.csr
│   ├── root-cert.pem
│   └── root-key.pem
├── setup-mesh.sh
└── topology.yaml
```

## intermediate-ca

为每个集群生成中间ca证书。从 `setup-mesh.sh` 脚本里能发现生成中间ca的方式 `make "intermediate-${CONTEXT}-certs"`

```sh
$ cd certs/
$ make "intermediate-cluster1-certs"
generating intermediate-cluster1/ca-key.pem
Generating RSA private key, 4096 bit long modulus
.......++
.................................++
e is 65537 (0x10001)
generating intermediate-cluster1/cluster-ca.csr
generating intermediate-cluster1/ca-cert.pem
Signature ok
subject=/O=Istio/CN=Intermediate CA/L=intermediate-cluster1
Getting CA Private Key
generating intermediate-cluster1/cert-chain.pem
Citadel inputs stored in intermediate-cluster1/

$ tree .
.
├── base.yaml
├── certs
│   ├── Makefile
│   ├── intermediate-cluster1
│   │   ├── ca-cert.pem
│   │   ├── ca-key.pem
│   │   ├── cert-chain.pem
│   │   ├── cluster-ca.csr
│   │   ├── intermediate.conf
│   │   └── root-cert.pem
│   ├── root-ca.conf
│   ├── root-cert.csr
│   ├── root-cert.pem
│   └── root-key.pem
├── setup-mesh.sh
└── topology.yaml
```

## 关注所生成证书的源数据

从生成的证书看，root-ca是10年有效期，intermediate-ca 是两年。如果要调整证书的有效期，org等参数。Makefile提供了多个环境变量，可具体查看Makefile，下面列举几个

```
# variables: root CA
ROOTCA_DAYS ?= 3650
ROOTCA_KEYSZ ?= 4096
ROOTCA_ORG ?= Istio
ROOTCA_CN ?= Root CA
# Additional variables are defined in root-ca.conf target below.

#------------------------------------------------------------------------
# variables: intermediate CA (Citadel)
CITADEL_SERIAL ?= $(shell echo $$PPID) 	# certificate serial number (uses current PID)
CITADEL_DAYS ?= 730
CITADEL_KEYSZ ?= 4096
CITADEL_ORG ?= Istio
CITADEL_CN ?= Intermediate CA
CITADEL_SAN_DNS ?= localhost
```

## 一些和多集群相关的 IstioOperator 选项

### values.global.multiCluster.enabled

此选项打开时，会在manifest中新增3个资源，它们在 `manifests/charts/gateways/istio-ingress/templates/preconfigured.yaml` 中定义。我们需要关注该配置项与 `components.ingressGateways` 配置项的相互影响和具体行为 [configure-gateways](https://istio.io/docs/setup/install/istioctl/#configure-gateways)。当使用[Replicated control planes](https://istio.io/docs/setup/install/multicluster/gateways/)方式构建多集群网格，选定 gateway 入口时，需要特别关注。

1. `manifests/charts/gateways/istio-ingress` 会用于渲染 `components.ingressGateways` (profile 中) 定义的每一个ingressgateway。
1. `manifests/charts/gateways/istio-ingress/templates/preconfigured.yaml` 的逻辑是：`values.gateways["istio-ingressgateway"]` 存在定义就会生成3个 manifest 资源。即 `components.ingressGateways` 每定义一个，就会生成这3个资源，这3个资源的 namespace 与 `components.ingressGateways[i].namespace` 匹配，且 `selector` 始终匹配 `components.ingressGateways[i].name=istio-ingressgateway` 那个。所以可以理解为：当一个 namespace 创建了n个 ingressgateway 时，会生成n组3个资源，但它们都是相同的，最后每个 namespace 只会有一组资源。这里的设计和实现似乎是有缺陷的。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-multicluster-ingressgateway
  namespace: istio-system
  labels:
    app: istio-ingressgateway
    istio: ingressgateway
    release: istio
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "*.global"
    port:
      name: tls
      number: 15443
      protocol: TLS
    tls:
      mode: AUTO_PASSTHROUGH
---


apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: istio-multicluster-ingressgateway
  namespace: istio-system
  labels:
    app: istio-ingressgateway
    istio: ingressgateway
    release: istio
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: GATEWAY
      listener:
        portNumber: 15443
        filterChain:
          filter:
            name: "envoy.filters.network.sni_cluster"
    patch:
      operation: INSERT_AFTER
      value:
        name: "envoy.filters.network.tcp_cluster_rewrite"
        config:
          cluster_pattern: "\\.global$"
          cluster_replacement: ".svc.cluster.local"
---


apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: istio-multicluster-ingressgateway
  namespace: istio-system
  labels:
    app: istio-ingressgateway
    istio: ingressgateway
    release: istio
spec:
  host: "*.global"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

### values.global.controlPlaneSecurityEnabled

1. 从charts编排看，还有相当多的逻辑，主要与 istio-telemetry，istio-policy等相关(这些组件都基本废弃，[控制平面安全](https://istio.io/zh/news/releases/1.5.x/announcing-1.5/upgrade-notes/#control-plane-security) 描述了1.5版本的行为变更，文中还提到了1.5多集群安装不能工作相关）。通过设置该选项不同的值来生成 manifest做差异比较(profile:default)，差异仅体现在ConfigMap:istio-sidecar-injector 中的文件values 中的配置项 global.controlPlaneSecurityEnabled。
1. 通过跟踪 `istio-sidecar-injector`，它会被挂载到 istiod 容器路径 `/var/lib/istio/inject`，里面包含两个文件: config和values，config文件是一个模板文件，其中对 `values.global.controlPlaneSecurityEnabled` 的逻辑处理只有一处，如下所示，这个选项的逻辑分支永远也不会命中了，因为从1.5开始，`values.global.istiod.enabled`是true，从实际注入的sidecar上观察运行参数也是 `--controlPlaneAuthPolicy NONE`。从源码看，该选项仅在一处“validateFeatures” 有使用，没有更多应用层的逻辑与它相关。或许在未来，这个选项会被移除吧。

```
    {{- if .Values.global.istiod.enabled }}
    - --controlPlaneAuthPolicy
    - NONE
    {{- else if .Values.global.controlPlaneSecurityEnabled }}
    - --controlPlaneAuthPolicy
    - MUTUAL_TLS
    {{- else }}
    - --controlPlaneAuthPolicy
    - NONE
    {{- end }}
```


## 引用

[《多集群部署与管理》](https://www.servicemesher.com/istio-handbook/practice/multiple-cluster.html)
