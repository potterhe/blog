---
title: "Istio 1.5.0 实战：授权(Authorization)"
date: 2020-03-14T10:52:00+08:00
draft: true
tags: ["istio", "实战", "授权", "Authorization"]
slug: istio-1.5-in-action-authorization
---

在进行本节练习之前，先按照官方用例 [Authorization on Ingress Gateway](https://istio.io/docs/tasks/security/authorization/authz-ingress/) 完成 httpbin 的部署，httpbin 这个应用真的很好用。

```sh
$ kubectl apply -f samples/httpbin/httpbin.yaml -n test-mesh
$ kubectl apply -f samples/httpbin/httpbin-gateway.yaml -n test-mesh
```

从公网访问 http://your-lb-ip/，预期显示的是 httbin 的 swagger ui。`/ip`，`/anything`，`/headers` 等 restful api 会给我们调试验证带来极大的帮助。如下面的 `/anything` 请求，其行为是 “Returns anything that is passed to request”。

```sh
$ curl http://your-lb-ip/anything
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "your-lb-ip",
    "User-Agent": "curl/7.54.0",
    "X-B3-Parentspanid": "7d83460f9c0bb319",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "e2a3c9482108d0b5",
    "X-B3-Traceid": "149ddcb0970898197d83460f9c0bb319",
    "X-Envoy-Internal": "true",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/test-mesh/sa/httpbin;Hash=34b9847b00b649e3ab395fb895245443b063430cad251f0f5faa6bc67c85a59b;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  },
  "json": null,
  "method": "GET",
  "origin": "172.24.0.65",
  "url": "http://your-lb-ip/anything"
}
```

## 基于 IP 白名单的访问控制

基于 IP 白名单来实施访问控制，是一种很常规的保护网站的手法。我们尝试用它来保护 httpbin 只能被白名单的 IP 访问。从 httpbin 容器里获取客户端 IP。

```sh
$ curl http://your-lb-ip/ip
{
  "origin": "172.24.0.65"
}
```

上面的输出显示，从 httpbin 容器里取到的源 IP 是 172.24.0.65，这个 IP 显然不是客户端 IP，那这个 IP 来自何处呢？下面的解释是请教 tke 团队的 roc 获得的答复。

    externalTrafficPolicy 为 Cluster (默认值)时，转发到 endpoint 时，Service 的实现会做 snat，源 ip 就会变成节点上 cbr0 的 ip。

### 如何获取客户端 IP

1. [Source IP for Services with Type=NodePort](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-nodeport)
2. [保留客户端源 IP](https://kubernetes.io/zh/docs/tasks/access-application-cluster/create-external-load-balancer/#%E4%BF%9D%E7%95%99%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%BA%90-ip)
3. [使用 Source IP](https://kubernetes.io/zh/docs/tutorials/services/source-ip/)

小结以上文章。istio-ingressgateway 要获取客户端 IP，需要满足

- 设置 `service.spec.externalTrafficPolicy: Local`
- lb 的 backend 节点上需要有 istio-ingressgateway 的 Pod，这一个条件我们在 [Istio 1.5.0 实战：安装]({{< ref "istio-1.5-in-action-setup.md" >}}) 一节已经做过处理。（**这里请注意，如果要应用于大规模的集群，这里仍然有一些工作要做。特别是“通过设置 service.spec.externalTrafficPolicy 字段值为 Local ，故意导致健康检查失败 ( Local 的行为是：节点没有 Pod 会丢包) 来强制使没有 endpoints 的节点把自己从负载均衡流量的可选节点列表中删除”这个行为，有待验证，如果它是有效的，那我们需要保证这些节点上都有 istio-ingressgateway 的 Pod，这可能涉及反亲和性和更细致的处理**）。

在 override 文件中设置 `values.gateways.istio-ingressgateway.externalTrafficPolicy=Local` 后，应用到集群。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
  values:
    gateways:
      istio-ingressgateway:
        serviceAnnotations:
          # https://cloud.tencent.com/document/product/457/18210
          service.kubernetes.io/qcloud-loadbalancer-backends-label: company.com/ingressgateway=true
        nodeSelector:
          company.com/ingressgateway: true
        externalTrafficPolicy: Local
```

确认 svc/istio-ingressgateway 生效，在输出中确认如下文本。

```
$ kubectl describe svc istio-ingressgateway -n istio-system
...
External Traffic Policy:  Local
...
```

再次确认 httpbin 容器获取到的客户端 IP，确认是自己的 IP。

```sh
$ curl http://your-lb-ip/ip
{
  "origin": "110.183.160.24"
}
```

到此，从应用容器已经可以取到客户端的 IP，下面我们来应用“授权”策略。

## 授权策略 (Authorization Policy)

1. [Authorization Policy](https://istio.io/docs/reference/config/security/authorization-policy/)
2. [Authorization Policy Conditions](https://istio.io/docs/reference/config/security/conditions/)

