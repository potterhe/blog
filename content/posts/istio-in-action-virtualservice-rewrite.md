---
title: "Istio 实战：VirtualService rewrite"
date: 2020-07-01T10:00:00+08:00
draft: false
tags: ["istio", "实战", "VirtualService", "rewrite"]
slug: istio-in-action-virtualservice-rewrite
---

## 环境

- istio 1.6.3
- k8s 1.16.3

## 场景

需求与下述的两个用例完全一致：将请求的特定前缀删除后，再转发给后端应用，这是一个很普通的 rewrite 场景，下面的两个用例也给出了解决方案且有效。官方的文档[HTTPRewrite](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRewrite)，描述相当的简单，信息量很少，在排查问题的时候遇到困扰，好在找到了下面的两个用例，过程记录如下。

- [Rewrite url to the root in the gateway](https://discuss.istio.io/t/rewrite-url-to-the-root-in-the-gateway/2860)
- [istio: VirtualService rewrite to the root url](https://stackoverflow.com/questions/60658439/istio-virtualservice-rewrite-to-the-root-url)

## 目标

1. 请求分发是否符合预期
1. rewrite 是否符合预期，不同的istio-proxy(istio-ingressgateway, sidecar)在这个过程中承担的角色和行为。

## 过程

打开envoy的accessLog，重建 istio-ingressgateway 工作负载、应用工作负载，以让这些pod里的istio-proxy 应用日志选项。

```sh
$ kubectl -n istio-system edit cm istio
......
# Set accessLogFile to empty string to disable access log.
accessLogFile: "/dev/stdout" # 开启日志
......
```

以官方用例中的httpbin来做这个实验，只是对httpbin-gateway中的 VirtualService:httpbin 增加了rewrite选项，如下。这里要实现的意图与上文中 rewrite-url-to-root 的场景是一样的，解决方案也是使用的stackoverflow中描述的方案。

```
   gateways:
   - httpbin-gateway
   http:
+  - match:
+    - uri:
+        prefix: /api
+    rewrite:
+      uri: " "
+    route:
+    - destination:
+        host: httpbin
+        port:
+          number: 8000
   - route:
     - destination:
         host: httpbin
```

打开3个终端，观察行为。

终端1：观察 istio-ingressgateway 的访问日志

```sh
$ kubectl logs -f istio-ingressgateway-company-c-7bd9fd4c9-4wkmp -n rd-1
```

终端2: 观察应用pod里sidecar的访问日志

```sh
$ kubectl logs -f httpbin-779c54bf49-xv8qt -c istio-proxy -n rd-1
```

终端3，发起请求，查看应用容器里观察到的请求, httpbin的 `/anything` api 会返回请求本身的信息，能帮忙我们从应用的视角来观察请求。

```sh
$ curl https://httpbin.company.com/anything
```

### 实验

```sh
$ curl https://httpbin.company.com/api/anything
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.company.com",
    "User-Agent": "curl/7.54.0",
    "X-B3-Parentspanid": "a3281800932a2598",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "d32b68e4b2ac2f88",
    "X-B3-Traceid": "dd6294459b25fa8ca3281800932a2598",
    "X-Envoy-External-Address": "182.139.101.81",
    "X-Envoy-Original-Path": "/api/anything",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/rd-1/sa/httpbin;Hash=8f43054a3c39af4acdd173e64062b8cbca3535cd84da93d4b780933d4ec61760;Subject=\"\";URI=spiffe://cluster.local/ns/rd-1/sa/istio-ingressgateway-company-c-service-account"
  },
  "json": null,
  "method": "GET",
  "origin": "182.139.101.81",
  "url": "https://httpbin.company.com/anything"
}

# 注意上面响应报文的倒数第二行

#istio-ingressgateway 的访问日志

[2020-07-01T01:23:01.437Z] "GET /api/anything HTTP/2" 200 - "-" "-" 0 847 1 1 "182.139.101.81" "curl/7.54.0" "107f4bb2-d5ee-93da-9025-d530eb3c8cde" "httpbin.company.com" "192.168.224.9:80" outbound|8000||httpbin.rd-1.svc.cluster.local 192.168.224.179:53558 192.168.224.179:8443 182.139.101.81:21585 httpbin.company.com -

#sidecar 的访问日志, 注意请求的 path

[2020-07-01T01:23:01.439Z] "GET /api/anything HTTP/1.1" 200 - "-" "-" 0 847 1 1 "182.139.101.81" "curl/7.54.0" "107f4bb2-d5ee-93da-9025-d530eb3c8cde" "httpbin.company.com" "127.0.0.1:80" inbound|8000|http|httpbin.rd-1.svc.cluster.local 127.0.0.1:49940 192.168.224.9:80 182.139.101.81:0 outbound_.8000_._.httpbin.rd-1.svc.cluster.local default
```

## 小结

1. 注意响应报文的倒数第二行，应用观察到的url的path部分没有 `/api`，说明请求真实的被rewrite了，是符合预期的。
1. sidecar 的访问日志显示的请求path仍然是 `/api/anything`, 说明 istio-ingressgateway 并没有rewrite，rewrite是在sidecar上发生的。
1. 请求的匹配分发在是在 istio-ingressgateway 上进行的，由于上面的实现中 `/api` 的目标端仍然是httpbin本身，所以不能明显的感知，如果再做一个httpbin服务，就能感知。
1. 猜想这种策略应该是出于将负载分层的想法，istio-ingressgateway 作为网格的入口，服务于所有的应用，应尽可能的减少其负载，那些更繁杂的策略在各应用的sidecar上做，是合适的。

## 引用

[istio 数据面调试指南](https://zhonghua.io/2020/02/12/istio-debug-with-envoy-log/)
