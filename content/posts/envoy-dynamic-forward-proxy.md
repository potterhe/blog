---
title: "Envoy's dynamic forward proxy"
date: 2020-11-09T23:59:59+08:00
draft: false
tags: ["Envoy"]
slug: envoy-dynamic-forward-proxy
---

两个概念

- Reverse proxy 反向代理
- Forward proxy 正向代理

Envoy 文档 [HTTP dynamic forward proxy](https://www.envoyproxy.io/docs/envoy/v1.16.0/intro/arch_overview/http/http_proxy) 描述了“动态正向代理”是什么，设计意图等。[Configuration reference-Dynamic forward proxy](https://www.envoyproxy.io/docs/envoy/v1.16.0/configuration/http/http_filters/dynamic_forward_proxy_filter) 提供了一个样例配置，它在代理 http 请求时，是可以工作的，但代理 https 的请求不行，会返回一个404 错误。官方FAQ [Why is Envoy sending 404s to CONNECT requests?](https://www.envoyproxy.io/docs/envoy/v1.16.0/faq/debugging/why_is_envoy_404ing_connect_requests) 和 [Issue #11552](https://github.com/envoyproxy/envoy/issues/11552) 描述了 404 的问题。

```sh
$ curl -I -vvv https://www.baidu.com -x localhost:10000
* Rebuilt URL to: https://www.baidu.com/
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 10000 (#0)
* Establish HTTP proxy tunnel to www.baidu.com:443
> CONNECT www.baidu.com:443 HTTP/1.1
> Host: www.baidu.com:443
> User-Agent: curl/7.54.0
> Proxy-Connection: Keep-Alive
>
< HTTP/1.1 404 Not Found
HTTP/1.1 404 Not Found
< date: Tue, 10 Nov 2020 06:40:08 GMT
date: Tue, 10 Nov 2020 06:40:08 GMT
< server: envoy
server: envoy
< connection: close
connection: close
< content-length: 0
content-length: 0
<

* Received HTTP code 404 from proxy after CONNECT
* Closing connection 0
curl: (56) Received HTTP code 404 from proxy after CONNECT
```

通过以上的材料，我们基本了解了前因后果，但遗憾的是这些文章中提到的 `Upgrade`、`connect_matcher` 具体怎么配置官方文档并无相关用例，通过查阅[envoy api](https://github.com/envoyproxy/envoy/blob/v1.16.0/api/envoy/config/route/v3/route_components.proto),，得到如下可以工作的配置。

```sh
$ git diff
diff --git a/http/dynamic-forward-proxy/envoy.yaml b/http/dynamic-forward-proxy/envoy.yaml
index d9f51b8..ab3c40e 100644
--- a/http/dynamic-forward-proxy/envoy.yaml
+++ b/http/dynamic-forward-proxy/envoy.yaml
@@ -37,6 +37,13 @@ static_resources:
                   prefix: "/"
                 route:
                   cluster: dynamic_forward_proxy_cluster
+              - match:
+                  connect_matcher: {}
+                route:
+                  cluster: dynamic_forward_proxy_cluster
+                  upgrade_configs:
+                  - upgrade_type: CONNECT
+                    enabled: true
           http_filters:
           - name: envoy.filters.http.dynamic_forward_proxy
             typed_config:
@@ -58,10 +65,3 @@ static_resources:
         dns_cache_config:
           name: dynamic_forward_proxy_cache_config
           dns_lookup_family: V4_ONLY
-    transport_socket:
-      name: envoy.transport_sockets.tls
-      typed_config:
-        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
-        common_tls_context:
-          validation_context:
-            trusted_ca: {filename: /etc/ssl/certs/ca-certificates.crt}
```

## 引用

1. [理解HTTP CONNECT通道](https://joji.me/zh-cn/blog/the-http-connect-tunnel/) 这篇文章言简意赅地描述了 https 透明正向代理的实现原理。
1. [Haproxy proxy protocol](https://www.haproxy.org/download/1.9/doc/proxy-protocol.txt) 描述了一种代理协议。
