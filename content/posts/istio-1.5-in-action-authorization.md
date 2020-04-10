---
title: "Istio 1.5 实战：授权(Authorization)"
date: 2020-03-20T23:37:00+08:00
draft: false
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

## 基于 IP 的访问控制

基于 IP 来实施访问控制，是一种很常规的保护网站的手法。通过观测 httpbin 容器里的输出，确认客户端 IP。

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
  "origin": "105.133.10.12"
}
```

到此，从应用容器已经可以取到客户端的 IP，下面我们来应用“授权”策略。

## 授权策略 (Authorization Policy)

1. [Authorization Policy](https://istio.io/docs/reference/config/security/authorization-policy/)
1. [Authorization Policy Conditions](https://istio.io/docs/reference/config/security/conditions/)
1. [IP-based allow list and deny list](https://istio.io/docs/tasks/security/authorization/authz-ingress/#ip-based-allow-list-and-deny-list) 展示了 istio-ingressgateway 工作负载上的“基于 IP 的白名单和黑名单”的用法。
1. [spec.rules.from.source](https://istio.io/docs/reference/config/security/authorization-policy/#Source) 定义源端。如来源 IP。
1. [spec.rules.to.operation](https://istio.io/docs/reference/config/security/authorization-policy/#Operation) 定义目标端。如请求的域名。
1. [spec.rules.when](https://istio.io/docs/reference/config/security/authorization-policy/#Condition) 定义追加的条件。支持许多协议相关的属性，这里是[完整的属性列表](https://istio.io/docs/reference/config/security/conditions/)。

### 策略规则的行为

授权策略支持 allow 和 deny 策略，当 allow 和 deny 同时作用于工作负载时，deny 优先。下面是具体的规则：

1. 如果请求匹配任何 DENY 策略，拒绝请求。
1. 如果工作负载没有任何的 ALLOW 策略，放行请求。
1. 如果请求匹配任何 ALLOW 策略，放行请求。
1. 拒绝请求。

### 域名级：保护监控、观测组件

现在我们来编写一个策略，实现 grafana.company.com 这个域名只能被限定的来源 IP 访问。为了方便观测，我们用 httpbin 来作为测试桩。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: istio-ingressgateway-policy
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - from:
    - source:
       ipBlocks: ["105.133.10.12"]
    to:
    - operation:
       hosts:
       - grafana.company.com
```

#### 本地 IP 和 ipBlocks 匹配时

```sh
$ curl -I -H "Host: grafana.company.com" http://your-lb-ip/ip
HTTP/1.1 200 OK
server: istio-envoy
date: Fri, 20 Mar 2020 07:37:41 GMT
content-type: application/json
content-length: 33
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 5
```

#### 本地 IP 和 ipBlocks 不匹配时

```sh
$ curl -I -H "Host: grafana.company.com" http://your-lb-ip/ip
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Fri, 20 Mar 2020 07:37:30 GMT
server: istio-envoy

RBAC: access denied
```

现在我们可以把想要保护的域名都加入到策略规则中。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: istio-ingressgateway-policy
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - from:
    - source:
       ipBlocks: ["105.133.10.12"]
    to:
    - operation:
       hosts:
       - grafana.company.com
       - tracing.company.com
       - kiali.company.com
```

#### 访问其它不在授权策略里的域名

```sh
$ curl -I -H "Host: api.company.com" http://your-lb-ip/ip
HTTP/1.1 403 Forbidden
content-length: 19
content-type: text/plain
date: Fri, 20 Mar 2020 07:52:14 GMT
server: istio-envoy
```

响应报文和 “本地 IP 和 ipBlocks 不匹配时”一致，少了 “RBAC: access denied” 输出。这里的行为是符合“策略规则的行为”一节的描述的，如果对这有疑问，请仔细理解。为了放行 api.compony.com 我们需要显式的添加一条规则。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: istio-ingressgateway-policy
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - from:
    - source:
       ipBlocks: ["105.133.10.12"]
    to:
    - operation:
       hosts:
       - grafana.company.com
       - tracing.company.com
       - kiali.company.com
  - to:
    - operation:
       hosts:
       - api.company.com
```

## “访问控制”的其它选项

### 多 istio-ingressgateway

一般我们会将“面向用户的业务入口”和“企业内部的职能支撑系统入口”分开。对于前者，ToC 的业务一般不会设置访问控制，ToB 的业务有更大概率会有；后者的访问控制会更严格一些，且从这个入口进入的规则基本都是一致的——限定仅允许办公网或vpn来源访问，一般不会对某个具体的应用制定规则。Gateway 定义中通过 `gateway.spec.selector` 选择指定的网格入口。

### 在“应用层”实现访问控制

这可能会引发异议，因为按照服务网格的初衷，访问控制是网格的职能，应该对业务透明、无感知，这里却说在应用层去实现访问控制，有误导人的嫌疑。当然这种提法需要一些前提来支撑，这里更多不是技术层面的问题，而是一种权衡。

1. 越是靠近接入层（istio-ingressgateway），我们一般越想保持它的稳定，能不动就不动，虽然evony提供了很好的热加载能力，可以快速响应变更。
1. 变更的风险。在 istio-ingressgateway 上编写授权策略规则，在规则复杂的情况下，一旦授权策略出现了问题，影响是全接入层的。当然你可以说关注点错误了，应当去关注保证授权策略的正确性，通过测试，验证流程回归和覆盖。但从风险控制的角度，把影响控制在局部，仍然是值得考虑的选项。这些都需要根据业务的实际需求来权衡。比如：在“面向用户的业务入口”上，有一个应用（比如 openapi 这样的），会开放部分能力给第三方合作伙伴，但小厂限于安全方面的技术实力，出于风险控制，一般通过限制第三的来源IP + 应用层的认证和授权机制来控制，这种情况在应用层实现就好些，因为一般会有多个第三方合作伙伴，这些规则一般很多，会存在大量定制的需求，变更相对频繁；如果是“企业内部的职能支撑系统入口”，这种在 istio-ingressgateway 上添加规则更合理，因为这种场景，需求本身就是接入层级别的，在应用层实现反而不好，即使是这样，也建议在接入层保持规则的简单性。很多决策不是技术层的问题，是人的问题。
1.  应用层上编写授权策略会面临一些障碍，比如：客户端源 IP 就较麻烦，通过 httpbin 的 /headers api 测试，业务的 sidecar 感知到的 source.ip 有些复杂。它可能是 istio-ingressgateway 的 Pod 所在节点的 crb0 的 IP（Service 的实现做 snat）；根据网络层的实现，情况可能更复杂些。真实的客户端 IP 是通过 header `X-Envoy-External-Address` 传递的，授权策略通过的 `when` 配置项（request.headers）提供支持，但是字符串类型的。
1. 在应用前增加一层透明代理(如 nginx)来实现访问控制。理论上业务 Pod 里已经有一个 sidecar 了，不需要另一个“代理”，这里的出发点主要是：在非网络领域，envoy还是比较陌生，基本还是 nginx 使用得更为广泛，人们对nginx的配置都很熟悉，行为认知方面都积累了大量的经验。nginx 里对 header 进行变换是很轻松的事情，可以很容易的应用于 allow、deny 配置项。

## 小结

现在监控、观测组件得到了有限的保护，至少比裸奔安全了，后面我们通过“认证”机制来加固。
