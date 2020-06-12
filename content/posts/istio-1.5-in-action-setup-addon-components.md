---
title: "Istio 1.5 实战：“体验”监控，观测"
date: 2020-03-12T12:52:00+08:00
draft: false
tags: ["istio", "实战", "观测", "安装"]
slug: istio-1.5-in-action-setup-addon-components
---

**本文的内容已经过时，未来的发行版将删除addon，请详见官方的博客[Reworking our Addon Integrations](https://istio.io/latest/blog/2020/addon-rework/)，addon 整合已经不是社区未来的方向，所以可以极早向上游部署方式看齐，这是一个好的方向。**

本文将部署 Istio 的监控，观测附加组件 grafana、tracing、kiali 。标题中使用了“体验”这个词，所以本文所做的练习**只是用于体验，切不可用于生产**，且行文的表述都是基于“体验”这个前提，体验完后，请务必清理。这种建议主要基于以下考虑:

- 安全。在没有应用“认证”、“授权”机制前，这些服务裸奔在公网是危险的，即使有的系统有帐密。
- 存储。这些服务都是有状态的，涉及状态数据的存储，如：promethues、grafana、kiali、tracing。这些 Pod 一旦重建，数据就会丢失。一般地，需要我们适配 k8s 标准化过的存储机制，如：storageClass、pv、pvc等。

## 组件部署

查看支持的 addonComponents，以及默认的配置值。

```sh
$ istioctl profile dump --config-path addonComponents
2020-06-10T01:37:35.653332Z	info	proto: tag has too few fields: "-"
grafana:
  enabled: false
  k8s:
    replicaCount: 1
istiocoredns:
  enabled: false
kiali:
  enabled: false
  k8s:
    replicaCount: 1
prometheus:
  enabled: true
  k8s:
    replicaCount: 1
tracing:
  enabled: false
```


### grafana

在 override 文件中添加 `addonComponents.grafana.enabled=true` 项后，应用到集群。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
  addonComponents:
    grafana:
      enabled: true
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        serviceAnnotations:
          # https://cloud.tencent.com/document/product/457/18210
          service.kubernetes.io/qcloud-loadbalancer-backends-label: company.com/ingressgateway=true
        nodeSelector:
          company.com/ingressgateway: "true"
        service:
          ports:
            # 公网lb控制暴露的端口.
          - name: http2
            port: 80
            targetPort: 80
          - name: https
            port: 443
```

查看现场，预期会增加以下对象

```sh
$ kubectl get all -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/grafana-5dc9c87cdf-j6ml8                1/1     Running   0          3d9h

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                    AGE
service/grafana                     ClusterIP      172.24.31.223   <none>          3000/TCP                                                   3d9h

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana                1/1     1            1           3d9h

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-5dc9c87cdf                1         1         1       3d9h
```

定义 Gateway 暴露服务。保存以下内容到 istio-addon-components-gateway.yaml。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-addon-components-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "grafana.company.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istio-grafana
spec:
  hosts:
  - "grafana.company.com"
  gateways:
  - istio-addon-components-gateway
  http:
  - route:
    - destination:
        host: grafana
        port:
          number: 3000
```

应用到集群。注意：Gateway 对象要和工作负载定义在同一个 namespace。

```
$ kubectl apply -f istio-addon-components-gateway.yaml -n istio-system
```

体验。绑定 hosts 后，浏览器访问 http://grafana.company.com ，完成后执行清理。
```
$ kubectl delete -f istio-addon-components-gateway.yaml -n istio-system
```

### tracing

同 grafana 类似的部署流程，修改 override 文件，添加 `addonComponents.tracing.enabled=true`，`values.pilot.traceSampling=100` 这个选项是为提升采样率到100%，以采样更多数据，体验tracing的功能，生产默认值是1。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
  addonComponents:
    grafana:
      enabled: true
    tracing:
      enabled: true
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        serviceAnnotations:
          # https://cloud.tencent.com/document/product/457/18210
          service.kubernetes.io/qcloud-loadbalancer-backends-label: company.com/ingressgateway=true
        nodeSelector:
          company.com/ingressgateway: "true"
        service:
          ports:
            # 公网lb控制暴露的端口.
          - name: http2
            port: 80
            targetPort: 80
          - name: https
            port: 443
  values:
    pilot:
      traceSampling: 100
```

查看现场，预期会新增以下对象。

```sh
$ kubectl get all -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istio-tracing-8648c56f59-qmcgk          1/1     Running   0          3d9h

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                    AGE
service/jaeger-agent                ClusterIP      None            <none>          5775/UDP,6831/UDP,6832/UDP                                 3d9h
service/jaeger-collector            ClusterIP      172.24.29.164   <none>          14267/TCP,14268/TCP,14250/TCP                              3d9h
service/jaeger-collector-headless   ClusterIP      None            <none>          14250/TCP                                                  3d9h
service/jaeger-query                ClusterIP      172.24.30.229   <none>          16686/TCP                                                  3d9h
service/tracing                     ClusterIP      172.24.30.249   <none>          80/TCP                                                     3d9h
service/zipkin                      ClusterIP      172.24.31.112   <none>          9411/TCP                                                   3d9h

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-tracing          1/1     1            1           3d9h

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/istio-tracing-8648c56f59          1         1         1       3d9h
```

定义 Gateway 暴露服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-addon-components-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "grafana.company.com"
    - "tracing.company.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istio-grafana
spec:
  hosts:
  - "grafana.company.com"
  gateways:
  - istio-addon-components-gateway
  http:
  - route:
    - destination:
        host: grafana
        port:
          number: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istio-tracing
spec:
  hosts:
  - "tracing.company.com"
  gateways:
  - istio-addon-components-gateway
  http:
  - route:
    - destination:
        host: tracing
        port:
          number: 80
```

体验之前，请执行一些 bookinfo 示例页面的访问，以触发采样。清理步骤与 grafana 节描述的一致。

### kiali

kiali 有帐号、密码验证，登陆的帐号和密码保存在 k8s 的名为 kiali (名字可变，通过 `values.kiali.dashboard.secretName` 设置) 的 Secret 对象中，这个 Secret 对象通过 volume 机制挂载到 Pod，所以需要事先创建，变更内容后，需要重建 Pod。通过设置 `values.kiali.createDemoSecret=true` 可以自动创建一个 Secret 对象（帐号，密码均为 “admin”），请谨慎决策。Secret 对象准备好后，在 override 文件中添加 `addonComponents.kiali.enabled=true` 项，应用到集群。

[使用 Kiali 观察您的服务网格](https://aws.amazon.com/cn/blogs/china/observe-service-mesh-kiali/)

```yaml
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
  addonComponents:
    grafana:
      enabled: true
    kiali:
      enabled: true
    tracing:
      enabled: true
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        serviceAnnotations:
          # https://cloud.tencent.com/document/product/457/18210
          service.kubernetes.io/qcloud-loadbalancer-backends-label: company.com/ingressgateway=true
        nodeSelector:
          company.com/ingressgateway: "true"
        service:
          ports:
            # 公网lb控制暴露的端口.
          - name: http2
            port: 80
            targetPort: 80
          - name: https
            port: 443
  values:
    pilot:
      traceSampling: 100
```

查看现场，预期会新增以下对象。

```sh
$ kubectl get all -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/kiali-7469f7b56d-l55q4                  1/1     Running   0          3d5h

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                    AGE
service/kiali                       ClusterIP      172.24.29.8     <none>          20001/TCP                                                  3d9h

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kiali                  1/1     1            1           3d9h

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/kiali-7469f7b56d                  1         1         1       3d9h
```

定义 Gateway 暴露服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-addon-components-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "grafana.company.com"
    - "tracing.company.com"
    - "kiali.company.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istio-grafana
spec:
  hosts:
  - "grafana.company.com"
  gateways:
  - istio-addon-components-gateway
  http:
  - route:
    - destination:
        host: grafana
        port:
          number: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istio-tracing
spec:
  hosts:
  - "tracing.company.com"
  gateways:
  - istio-addon-components-gateway
  http:
  - route:
    - destination:
        host: tracing
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: istio-kiali
spec:
  hosts:
  - "kiali.company.com"
  gateways:
  - istio-addon-components-gateway
  http:
  - route:
    - destination:
        host: kiali
        port:
          number: 20001
```

体验，清理步骤与 grafana 节描述的一致。

### prometheus

`addonComponents.prometheus.enabled` 默认是开启的，通过查看charts编排和实际pod的运行参数，这个服务基本不是生产级的，表现在

- `--storage.tsdb.retention=6h`，只会保留近6小时的数据，当然这个值是可以调整的。
- 数据持久化方面，数据是落地在容器里的, 并没有挂载pvc，charts里也没有提供相关的配置项支持。即容器重建的场景，数据将丢失。

综上，实际生产中，我们需要关闭它，用自建的prometheus-operator来代替，这个涉及多个方面，如kaili有依赖prometheus。

## 小结

很粗放的部署了以“体验”为目的的观测服务，后续我们将应用“认证”、“授权”机制来保护这些服务，Gateway 支持 https 服务。
