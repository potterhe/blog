---
title: "Istio 实战：认证--RequestAuthentication"
date: 2020-03-20T23:40:00+08:00
draft: false
tags: ["istio", "实战", "认证", "Authentication", "RequestAuthentication"]
slug: istio-in-action-request-authentication
toc: true
---

根据官方文档：[认证](https://istio.io/docs/concepts/security/#authentication) 章节的描述，Istio 提供两种认证机制(PeerAuthentication，RequestAuthentication)，PeerAuthentication 解决工作负载间的问题，RequestAuthentication 解决用户端的问题。本文关注用 RequestAuthentication 来保护“裸”应用。以下是需要先从官网了解的相关知识：

- [认证](https://istio.io/docs/concepts/security/#authentication)
- [Request authentication](https://istio.io/docs/concepts/security/#request-authentication)

## 环境

istio 1.5.2

## RequestAuthentication

先看看 manifest 长什么样子，下面是官网的一个样例，主要由两个字段构成 `selector`，`jwtRules`。

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/master/security/tools/jwt/samples/jwks.json"
```

- [Reference: Request authentication](https://istio.io/docs/reference/config/security/request_authentication/) 描述了manifest 的完整定义。
- [Reference: JWTRule](https://istio.io/docs/reference/config/security/jwt/) 描述了 `jwtRules` 的定义。

### Selector

`selector` 通过 label 机制选择适用该策略的目标工作负载。

1. 您可以将JWT策略添加到入口网关（上面的样例就是）。 这通常用于为绑定到网关的所有服务而不是单个服务定义JWT策略。下面是[官方文档](https://istio.io/docs/tasks/security/authentication/authn-policy/#end-user-authentication)的原文

    You can also add a JWT policy to an ingress gateway (e.g., service istio-ingressgateway.istio-system.svc.cluster.local). This is often used to define a JWT policy for all services bound to the gateway, instead of for individual services.

1. 一般地，更建议在特定的工作负载上绑定特定的 JWT 策略，除非它满足第1条的场景。这相当于 ingressgateway 是一个透明代理，在工作负载的 sidecar 上执行策略检查.

### JWTRule

`jwtRules` 字段是一个数组，这意味着我们可以为匹配的工作负载指定多个 JWT 策略。当我们要接入多个 OpenID Connect Provider 时，这是必要的。官方文档对有多个规则时，如何标记 token 位置；以及当多个有效token时的行为有明确说明。这里对部分字段作说明：

- `fromHeaders` 声明 JWT 在 header 中的位置、前缀。
- `fromParams` 声明 JWT 在 queryString 中参数的名字。
- `issuer` 限定 JWT 里的 `iss` claim，不匹配则拒绝访问。
- `audiences` 限定 JWT 里的 `aud` claim，不匹配则拒绝访问。
- `jwksUri`、`jwks` 声明验证 JWT 有效性的证书或下载证书的 uri。
- `outputPayloadToHeader` 如果工作负载要使用 JWT.payload 中的数据，则这是必要的。对于大多数业务系统，应该都需要配置它。
- `forwardOriginalToken` 一般是不必要的

#### JWT

由于 RequestAuthentication 是基于 JWT 的，掌握它是必要的，以下的材料对掌握 JWT 很有用

- [rfc7519](https://tools.ietf.org/html/rfc7519)
- 《jwt-handbook》一本免费的书，比 rfc 更友好些。
- [jwt.io](https://jwt.io/)

## 实战{#in-action}

### 构建 JWT{#gen-jwt}

按照官方 Task [Authentication Policy](https://istio.io/docs/tasks/security/authentication/authn-policy/#end-user-authentication) 里的描述，下载 gen-jwt.py 和 key.pem 这两个文件。然后我们构建 python 环境并使用它们来构建 JWT，后续的实操需要构建特定的 JWT。

#### 使用 pyenv 构建 python 环境

这里的操作是以 Mac OS X 来演示的，其它平台是一致的。

```sh
$ brew install pyenv
$ pyenv versions
$ pyenv install -l
$ pyenv install 3.7.5
$ pyenv versions
* system (set by /Users/xxx/.pyenv/version)
  3.7.5
```

用 virtualenv 构建一个虚拟环境

```sh
$ brew install pyenv-virtualenv
$ pyenv virtualenv 3.7.5 env-3.7.5
$ pyenv activate env-3.7.5

Failed to activate virtualenv.

Perhaps pyenv-virtualenv has not been loaded into your shell properly.
Please restart current shell and try again.
```

添加以下内容到shell 的 profile，比如 .bash_profile，重开 term，以触发生效。

```sh
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

再次激活目标env，安装 jwcrypto

```sh
$ pyenv activate env-3.7.5
pyenv-virtualenv: prompt changing will be removed from future release. configure `export PYENV_VIRTUALENV_DISABLE_PROMPT=1' to simulate the behavior.

(env-3.7.5) username@host: curr_dir$ pip install jwcrypto
Collecting jwcrypto
...
```

由于 gen-jwt.py 的首行指定的 `#!/usr/bin/python` 所以 virtualenv 不生效。修改为 `#!/usr/bin/env python`，不出意外，已经可以成功生成 JWT 了。

```sh
(env-3.7.5) username@host: curr_dir$ ./gen-jwt.py ./key.pem --expire 5
eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE1ODY0MTUzODEsImlhdCI6MTU4NjQxNTM3NiwiaXNzIjoidGVzdGluZ0BzZWN1cmUuaXN0aW8uaW8iLCJzdWIiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyJ9.sMCeWdl2BgP4nttpOcKD6cW4Z2rficQyFsyFvJtjPSZ8XVZySdmoXExVofjlIQhxDps0IKgJZmsp3j4rm1_koWkYCzbhfkOpm-UFUXkgK3Z2oZHkaFxhKqjIgV2TGdnjo4e9eIhwXpcXgS6P5zepCZj4p4uCc6pzYLI9oQL3SmB06aXoCIV2zjPql1IU1ydEp7IMNvH3FCHqKQeMBONIcicz_12FJRZJyKEVFeoIIncCHJxxt_1oifomyXRApdalYmtX--0fQ1BFEvfj96Sxu0NbEI7szGWVBpvf9GS-YRIbGOsLf3uPlVwDZp3yKK-n9pwQRDRO0dpVIrYzqw0dAQ
```

### 观察当前的状态{#describe-curr-environment}

确认系统中没有 RequestAuthentication 策略

```sh
$ kubectl get requestauthentications.security.istio.io --all-namespaces
No resources found.
```

确认 httpbin 服务的暴露方式.

```sh
$ kubectl get virtualservices.networking.istio.io -n test-mesh
NAME       GATEWAYS             HOSTS   AGE
httpbin    [httpbin-gateway]    [*]     25d
```

验证示例服务上没有适用的 DestinationRule。 您可以通过检查现有目标规则的host：值并确保它们不匹配。

```sh
$ kubectl get destinationrules.networking.istio.io --all-namespaces -o yaml | grep "host:"
```

### 构建测试桩

在官方用例的基础上增加了 `outputPayloadToHeader`、`forwardOriginalToken` 以便观察。**生产环境请根据实际情况决定是否加这些选项，这里加上只是便于观测**。

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/master/security/tools/jwt/samples/jwks.json"
    outputPayloadToHeader: "X-Jwt-Playload"
    forwardOriginalToken: true
```

### 不提供 JWT{#none-jwt-test}

```sh
$ export INGRESS_HOST=your-lb-ip
$ curl $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
200
```

既然定义了认证策略，为什么不提供 JWT 会允许访问? 这个行为需要注意，可能和预期的认知有差异，这是因为认证策略定义的是认证的行为，当没有提供 JWT 时，它并不能断言认证成功或失败，因为没有进行认证的断言。如果想实现必须提供有效的 JWT 才允许访问，则需要通过在“授权策略”中指定必须存在主体，具体请了解“授权策略”，这里就不展开了。

### 提供无效的 JWT{#invalid-jwt-test}

```sh
$ curl --header "Authorization: Bearer deadbeef" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
401
```

拒绝访问，返回 `401 Unauthorized`

### 提供有效的 JWT{#valid-jwt-test}

```sh
$ TOKEN=$(./gen-jwt.py ./key.pem --expire 5)
$ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
200
```

### 限定 issuer{#issuer-test}

这是一个声明，需由实现来保证它的行为正确，这里的实现是 sidecar

```sh
$ TOKEN=$(./gen-jwt.py ./key.pem --iss=testing1@secure.istio.io --expire=5)
$ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers
Jwt issuer is not configured
```

当提供的 JWT 的 iss 不在配置中时，响应 `401 Unauthorized`

### 限定 audiences{#audiences-test}

在认证策略中增加 `audiences` 后，应用到集群。

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/master/security/tools/jwt/samples/jwks.json"
    outputPayloadToHeader: "X-Jwt-Playload"
    forwardOriginalToken: true
    audiences:
    - bar
```

构造不匹配的 `aud` 的 JWT，响应403，拒绝访问

```sh
$ TOKEN=$(./gen-jwt.py ./key.pem --expire 50 --aud foo)
$ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
403

$ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers
Audiences in Jwt are not allowed
```

构造匹配的 `aud` 的 JWT，响应200，允许访问

```sh
$ TOKEN=$(./gen-jwt.py ./key.pem --expire 50 --aud foo,bar)

$ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"
200
```

### 用户注入 `outputPayloadToHeader` 指定的header{#injection-outputpayloadtoheader}

```sh
$ TOKEN=$(./gen-jwt.py ./key.pem --expire=50 --aud=foo,bar)
$ curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers
{
  "headers": {
    "Accept": "*/*",
    "Authorization": "Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IkRIRmJwb0lVcXJZOHQyenBBMnFYZkNtcjVWTzVaRXI0UnpIVV8tZW52dlEiLCJ0eXAiOiJKV1QifQ.eyJhdWQiOlsiZm9vIiwiYmFyIl0sImV4cCI6MTU4NjUwODA0NSwiaWF0IjoxNTg2NTA3OTk1LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InRlc3RpbmdAc2VjdXJlLmlzdGlvLmlvIn0.QusqbGZ0JAMmSzqpfQ2Nv-kC-ee6kHZiRdHq2sVs86RHtrWxLDpGY8VmKys2fpgIggCqRe7BhT_Uz6FvEVO3rPUUzkyBvCGHL_3UY9HaeECpHqiWIUyNmzN-w0_oXAc4lUWiXhlHon6tHUzu2qVsXPh2F4ULMabxtW3EC3jBV-aDMOd8aR1rXN88uLw9hIpx6msVubSqYfkpBmx3lnRL4gOlVHD9ONUOrhIcZkL9vtolul1S8mzdRuypj8EZtVe4uqSxw-1fOWhqlWSPv6ebsNNn62gOQgOqiURQqneXJ3SAcUwqXS5zG2LUJK9bZRDv5A76mEyR1Qg1Sc7Y0HSmaQ",
    "Content-Length": "0",
    "Host": "your-lb-ip",
    "User-Agent": "curl/7.54.0",
    "X-B3-Parentspanid": "daaaa53b4410de2e",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "fb2526175d8f0784",
    "X-B3-Traceid": "aacdc33418f16c95daaaa53b4410de2e",
    "X-Envoy-External-Address": "182.138.102.82",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/test-mesh/sa/httpbin;Hash=2ca8d1110b113d2bc48e1847254e3588feed9863b69a9bd6ee4fa9a8eacf56a4;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account",
    "X-Jwt-Playload": "eyJhdWQiOlsiZm9vIiwiYmFyIl0sImV4cCI6MTU4NjUwODA0NSwiaWF0IjoxNTg2NTA3OTk1LCJpc3MiOiJ0ZXN0aW5nQHNlY3VyZS5pc3Rpby5pbyIsInN1YiI6InRlc3RpbmdAc2VjdXJlLmlzdGlvLmlvIn0"
  }
}

$ curl --header "X-Jwt-Playload: xxx" $INGRESS_HOST/headers -s|grep Jwt
```

结论，用户请求不能注入 `outputPayloadToHeader` 指定的 header。

### 声明多个 JWT 规则

todo

### 限定必须提供有效的 JWT

[Task: Require a valid token](https://istio.io/docs/tasks/security/authentication/authn-policy/#require-a-valid-token) 描述了实现。

## 构建私有的 JWK

`gen-jwt.py` 工具提供了生成私有 JWK 的功能，文档描述在 [Regenerate private key and JWKS (for developer use only)](https://github.com/istio/istio/blob/master/security/tools/jwt/samples/README.md#regenerate-private-key-and-jwks-for-developer-use-only)

以下是用 `gen-jwt.py` 脚本生成 jwks 的方式，istio 的 README.md 文件里没有描述，从源码可以得到。
```sh
$ ./gen-jwt.py ./key.pem --jwks=./jwks.json
```

## 使用 RequestAuthentication 保护观测组件

由于观测组件是安装在 istio-system 命名空间的，而这个空间并不会注入 sidecar，所以当前只能在绑定了这些服务的 `istio:ingressgateway` 工作负载上应用认证策略。但这其实是不好的选择，建议用另一个 ingressgateway，与业务入口分离。

### 认证策略

移除 istio-ingressgateway 上的 RequestAuthentication 后，应用以下认证和授权策略。

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "istio-addon-components-jwt"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwks: '{ "keys":[ {"e":"AQAB","kid":"8CzPCO8ERS-yWH-tGmhnFj6irQ4LxqSNMA8jCVsQcc0","kty":"RSA","n":"uaCoiaE-FgXPhEI7YQHy-wyd_j0bVzbW1YM6nFnPcXpRSUFoToj8i2IsJ3cMyXN9mifGQIsw_LgojLfYOr2iTEDtPTYz4pX1O1NCn8n5tzyARjAGcwpCNoJ-dyMCRpWcBwj6Ro12lEdhDMG1i2EluUCUM6OELZMaAeG6qbXH1h1I5RzzwIrdaQe5eCAZTDMsBLi_3qzAUtEWRm5xC7JERBI25MLileZU8VkTNlhkjAIS6mgytK4Op7nd04GynhcjMPT8YEYDcj8KvYwgRM__dQK2H6XE3rk8wnNPgAd3JstkKSimjLtwR50_xGieQD1ZEXT9p-aLtWx1_f2EUBOm-Q"}]}'
    fromHeaders:
    - name: X-My-Authorization
      prefix: "Bearer "
    outputPayloadToHeader: "X-Jwt-Playload"

---
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: istio-addon-components-ap
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
    to:
    - operation:
       hosts:
       - tracing.company.com
       - grafana-istio.company.com
       - tracing.company.com
```

现在我们可以放心的开放这些服务了。

## 认证与授权

### 认证通过的信息怎么传递给“授权”机制

以下官方文档描述了少许细节。

1. [认证架构](https://istio.io/zh/docs/concepts/security/#authentication-architecture)
1. [Principals](https://istio.io/zh/docs/concepts/security/#principals)
1. [requestPrincipals](https://istio.io/docs/reference/config/security/authorization-policy/#Source)。`Source.requestPrincipals` 的描述有 “Optional. A list of request identities (i.e. “iss/sub” claims), which matches to the “request.auth.principal” attribute”

## 浏览器注入header

[Chrome 的 Modify Header Value 扩展](https://chrome.google.com/webstore/detail/modify-header-value-http/cbdibdfhahmknbkkojljfncpnhmacdek/related) 允许给指定的站点注入 header，我们可以方便的注入 `Authorization` 头

## 小结

通过控制颁发的 JWT 和配置认证策略，我们可以在不修改应用的情况下，实现一定粒度的访问控制。
