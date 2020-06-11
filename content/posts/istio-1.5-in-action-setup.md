---
title: "Istio 1.5 实战：安装"
date: 2020-03-11T23:59:50+08:00
draft: false
tags: ["istio", "1.5", "实战", "安装"]
slug: istio-1.5-in-action-setup
---

istio 1.5.0 于北京时间2020年3月6日发布，该版本在架构上有很大的变化([Istio 1.5 新特性解读](https://www.servicemesher.com/blog/istio-1-5-explanation/))，做出这些改变的原因及其重大意义这里不做赘述，有大量的文章做了阐述，本文聚焦于探索 istio-1.5.0 的行为。文中的练习是在腾讯云的容器服务 tke-1.16.3 上进行的，会涉及到一些云平台相关的实现。

## 安装istio

使用 istioctl 安装是官方推荐的方式([Customizable Install with Istioctl](https://istio.io/docs/setup/install/istioctl/))，helm 安装方式已经被标记为 deprecated，且已经不支持 helm3。istioctl 安装实践，一般选择一个profile（生产环境，社区推荐基于 default），然后用自定义的值进行覆盖，一般而言，我们只需要编写这个 override 文件(这里的文件名是 profile-override-default.yaml)，不需要修改 charts （当然charts是可以被修改的）。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
```

另一面，override 文件中的项，以及项的节点层次怎么找呢？建议两种方式：1，通过 `istioctl profile dump default` 命令可以打印一些，但不够全；2，查看 charts （install/kubernetes/operator/charts/） 里各子项目的 values 文件定义，这种方式最可靠，对值的行为还可以辅以 templates 文件更深入的了解，推荐这种。

有了 override 文件后，就可以使用下面的命令生成 manifest，manifest 是 k8s 的定义。建议把 override 文件和生成的 manifest 文件都通过版本控制工具维护起来，在每次对 override 操作后，对比前后的差异。

```sh
$ istioctl manifest generate -f profile-override-default.yaml  >  manifest-override-default.yaml
```

如果一切准备好后，就可以执行安装到集群操作，下面的两条命令是一样的效果。但这里我们先不操作，还有一些细节。

```sh
$ istioctl manifest apply -f profile-override-default.yaml
$ kubectl apply -f manifest-override-default.yaml
```

### istio-ingressgateway

istio-ingressgateway 是网格的入口，其实现是一个 k8s 的 Service，云上一般我们只需要将其类型声明为 LoadBalancer，就可以创建一个公网的lb作为公网入口。

### 腾讯云的实现

- 腾讯云 LoadBalancer 类型的 Service 有以下的实现细节：声明为 LoadBalancer 类型的 Service 会以 NodePort 相同行为的方式暴露(类型还是LoadBalancer, 只是行为同NodePort，如果你去查看集群中 Service 的 yaml文件，会发现每个端口都有NodePort声明)，lb自动挂载集群所有（也支持node label 指定）节点作为后端，Service 的端口绑定到节点的 NodePort 端口。这些都是自动的，行为是透明的。
- [腾讯云-容器服务-创建 Service](https://cloud.tencent.com/document/product/457/18210) 描述了腾讯云定义的 annotations, 实现了多种行为：“公网服务”，“内网服务”，“指定 Loadbalance 只绑定指定节点”等。腾讯云公网lb只会把公网流量代理到“购买有公网带宽”的节点，创建公网的 istio-ingressgateway 时，我们用到了这些 annotations，我们只希望购买了公网带宽的机器挂载到公网lb后端。
- tke支持istio-cni

通过覆盖 serviceAnnotations, nodeSelector和给公网带宽节点打上 label，实现了以下行为。且这些行为是保留客户端IP的必要条件，这是后话。

- 公网lb只挂载有公网带宽的节点作为后端。
- istio-ingressgateway 的 Pod 只部署在有公网带宽的节点上。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
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
```

给购买了公网带宽的 node 打上 label

```sh
$ kubectl label node 172.21.128.82 company.com/ingressgateway=true
```

### 安装到集群

```
$ istioctl manifest apply -f profile-override-default.yaml
proto: tag has too few fields: "-"
Detected that your cluster does not support third party JWT authentication. Falling back to less secure first party JWT. See https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens for details.
- Applying manifest for component Base...
✔ Finished applying manifest for component Base.
- Applying manifest for component Pilot...
✔ Finished applying manifest for component Pilot.
  Waiting for resources to become ready...
  Waiting for resources to become ready...
  Waiting for resources to become ready...
  Waiting for resources to become ready...
  Waiting for resources to become ready...
  Waiting for resources to become ready...
  Waiting for resources to become ready...
- Applying manifest for component AddonComponents...
- Applying manifest for component Cni...
- Applying manifest for component IngressGateways...
✔ Finished applying manifest for component IngressGateways.
✔ Finished applying manifest for component AddonComponents.
✔ Finished applying manifest for component Cni.


✔ Installation complete
```

### 确认安装现场

```
$ kubectl get crd |grep istio
adapters.config.istio.io                   2020-03-09T16:07:39Z
attributemanifests.config.istio.io         2020-03-09T16:07:39Z
authorizationpolicies.security.istio.io    2020-03-09T16:07:39Z
clusterrbacconfigs.rbac.istio.io           2020-03-09T16:07:39Z
destinationrules.networking.istio.io       2020-03-09T16:07:39Z
envoyfilters.networking.istio.io           2020-03-09T16:07:39Z
gateways.networking.istio.io               2020-03-09T16:07:39Z
handlers.config.istio.io                   2020-03-09T16:07:39Z
httpapispecbindings.config.istio.io        2020-03-09T16:07:39Z
httpapispecs.config.istio.io               2020-03-09T16:07:39Z
instances.config.istio.io                  2020-03-09T16:07:39Z
meshpolicies.authentication.istio.io       2020-03-09T16:07:39Z
peerauthentications.security.istio.io      2020-03-09T16:07:41Z
policies.authentication.istio.io           2020-03-09T16:07:41Z
quotaspecbindings.config.istio.io          2020-03-09T16:07:41Z
quotaspecs.config.istio.io                 2020-03-09T16:07:41Z
rbacconfigs.rbac.istio.io                  2020-03-09T16:07:41Z
requestauthentications.security.istio.io   2020-03-09T16:07:41Z
rules.config.istio.io                      2020-03-09T16:07:41Z
serviceentries.networking.istio.io         2020-03-09T16:07:41Z
servicerolebindings.rbac.istio.io          2020-03-09T16:07:41Z
serviceroles.rbac.istio.io                 2020-03-09T16:07:41Z
sidecars.networking.istio.io               2020-03-09T16:07:42Z
templates.config.istio.io                  2020-03-09T16:07:42Z
virtualservices.networking.istio.io        2020-03-09T16:07:42Z

$ kubectl api-resources |grep istio
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
meshpolicies                                   authentication.istio.io        false        MeshPolicy
policies                                       authentication.istio.io        true         Policy
adapters                                       config.istio.io                true         adapter
attributemanifests                             config.istio.io                true         attributemanifest
handlers                                       config.istio.io                true         handler
httpapispecbindings                            config.istio.io                true         HTTPAPISpecBinding
httpapispecs                                   config.istio.io                true         HTTPAPISpec
instances                                      config.istio.io                true         instance
quotaspecbindings                              config.istio.io                true         QuotaSpecBinding
quotaspecs                                     config.istio.io                true         QuotaSpec
rules                                          config.istio.io                true         rule
templates                                      config.istio.io                true         template
destinationrules                  dr           networking.istio.io            true         DestinationRule
envoyfilters                                   networking.istio.io            true         EnvoyFilter
gateways                          gw           networking.istio.io            true         Gateway
serviceentries                    se           networking.istio.io            true         ServiceEntry
sidecars                                       networking.istio.io            true         Sidecar
virtualservices                   vs           networking.istio.io            true         VirtualService
clusterrbacconfigs                             rbac.istio.io                  false        ClusterRbacConfig
rbacconfigs                                    rbac.istio.io                  true         RbacConfig
servicerolebindings                            rbac.istio.io                  true         ServiceRoleBinding
serviceroles                                   rbac.istio.io                  true         ServiceRole
authorizationpolicies                          security.istio.io              true         AuthorizationPolicy
peerauthentications                            security.istio.io              true         PeerAuthentication
requestauthentications                         security.istio.io              true         RequestAuthentication

$ kubectl get all -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istio-ingressgateway-78ff7c44f4-sccbj   1/1     Running   0          10h
pod/istiod-7f7c6fcbfd-kjzp8                 1/1     Running   0          10h
pod/prometheus-8b96f989b-7xqgk              2/2     Running   0          10h

NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                                                                                      AGE
service/istio-ingressgateway   LoadBalancer   172.24.30.242   x.x.x.x         15020:30438/TCP,80:30552/TCP,443:30120/TCP,15029:31267/TCP,15030:30278/TCP,15031:32237/TCP,15032:31853/TCP,15443:30194/TCP   10h
service/istio-pilot            ClusterIP      172.24.31.229   <none>          15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP                                                                     10h
service/istiod                 ClusterIP      172.24.28.18    <none>          15012/TCP,443/TCP                                                                                                            10h
service/prometheus             ClusterIP      172.24.28.131   <none>          9090/TCP                                                                                                                     10h

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/istio-ingressgateway   1/1     1            1           10h
deployment.apps/istiod                 1/1     1            1           10h
deployment.apps/prometheus             1/1     1            1           10h

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/istio-ingressgateway-78ff7c44f4   1         1         1       10h
replicaset.apps/istiod-7f7c6fcbfd                 1         1         1       10h
replicaset.apps/prometheus-8b96f989b              1         1         1       10h

NAME                                                       REFERENCE                         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   4%/80%    1         5         1          10h
horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 0%/80%    1         5         1          10h
```

#### 再探 istio-ingressgateway

istio-ingressgateway 这个服务我们需要特别关注，我们知道这是一个公网入口，它暴露了非常多的端口(这些端口的name语义在 `istioctl profile dump default --config-path values.gateways.istio-ingressgateway.ports` 的输出中有显示)，我们需要确认这些端口的用途和风险，预期中我们只需要暴露80和443端口。特别地，15443 端口在多集群部署（[Replicated control planes](https://istio.io/docs/setup/install/multicluster/gateways/)）时作为集群间流量的入口。

```sh
$ kubectl exec -it istio-ingressgateway-78ff7c44f4-sccbj bash -n istio-system

root@istio-ingressgateway-78ff7c44f4-sccbj:/# ps -efww
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Mar09 ?        00:00:29 /usr/local/bin/pilot-agent proxy router --domain istio-system.svc.cluster.local --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --log_output_level=default:info --drainDuration 45s --parentShutdownDuration 1m0s --connectTimeout 10s --serviceCluster istio-ingressgateway --zipkinAddress zipkin.istio-system:9411 --proxyAdminPort 15000 --statusPort 15020 --controlPlaneAuthPolicy NONE --discoveryAddress istio-pilot.istio-system.svc:15012 --trust-domain=cluster.local
root        19     1  0 Mar09 ?        00:02:07 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster istio-ingressgateway --service-node router~172.24.0.68~istio-ingressgateway-78ff7c44f4-sccbj.istio-system~istio-system.svc.cluster.local --max-obj-name-len 189 --local-address-ip-version v4 --log-format [Envoy (Epoch 0)] [%Y-%m-%d %T.%e][%t][%l][%n] %v -l warning --component-log-level misc:error
root        59     0  0 03:28 pts/0    00:00:00 bash
root        68    59  0 03:28 pts/0    00:00:00 ps -efww

root@istio-ingressgateway-78ff7c44f4-sccbj:/# netstat -l
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:15090           0.0.0.0:*               LISTEN
tcp        0      0 localhost:15000         0.0.0.0:*               LISTEN
tcp6       0      0 [::]:15020              [::]:*                  LISTEN
Active UNIX domain sockets (only servers)
Proto RefCnt Flags       Type       State         I-Node   Path
unix  2      [ ACC ]     STREAM     LISTENING     236671   /var/run/ingress_gateway/sds
unix  2      [ ACC ]     STREAM     LISTENING     235977   /etc/istio/proxy/SDS

/# lsof -i:15090
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
envoy    19 root   32u  IPv4 236714      0t0  TCP *:15090 (LISTEN)
```

通过以上的观察

- Pod 的15020端口，pilot-agent 监听，就绪探针使用 path: /healthz/ready。
- Pod 的15090端口，envoy的prometheus metric 端口，metrics_path: /stats/prometheus，这从prometheus的配置文件和 istio-ingressgateway 的 Deployment 定义的端口name: http-envoy-prom 确认。Pod 内访问 `curl http://localhost:15090/stats/prometheus` 可以获取数据。
- 15000端口，`--proxyAdminPort` 已经很明确的标识了语义。
- Pod 无443端口监听。因为还没有添加port 443 的 Gateway 对象。
- 其它 Service 上的端口在 Pod 上没有监听。通过设置 `values.gateways.istio-ingressgateway.telemetry_addon_gateways.tracing_gateway.enabled=true` 等配置项簇，可以启用这些端口，本质上是定义了一批 Gateway 对象。**但强烈不建议这么做，监控、观测的系统必须要做足够的访问控制，暴露这些系统也可以在80，443端口上通过域名来实现，但仍需要做到足够的访问控制**。访问控制是服务网格的重要能力，在 istio-1.5 之前的版本中，由 mixer 负责，从1.5开始已经明确表示要废弃，新机制在[security](https://istio.io/docs/concepts/security/)章节进行了描述。

调整 override 选项后，应用到集群。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
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

这里请注意service、serviceAnnotations的节点层次。

```
$ istioctl manifest apply -f profile-override-default.yaml
proto: tag has too few fields: "-"
Detected that your cluster does not support third party JWT authentication. Falling back to less secure first party JWT. See https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens for details.
- Applying manifest for component Base...
✔ Finished applying manifest for component Base.
- Applying manifest for component Pilot...
✔ Finished applying manifest for component Pilot.
- Applying manifest for component Cni...
- Applying manifest for component IngressGateways...
- Applying manifest for component AddonComponents...
✔ Finished applying manifest for component IngressGateways.
✔ Finished applying manifest for component Cni.
✔ Finished applying manifest for component AddonComponents.


✔ Installation complete
```

再次确认安装现场

```
$ kubectl get svc -n istio-system |grep istio-ingressgateway
istio-ingressgateway        LoadBalancer   172.24.30.242   x.x.x.x   80:30552/TCP,443:30120/TCP                                 2d12h
```

从公网lb访问 http://your-lb-ip/，会得到404响应。

## 部署 bookinfo

创建新的 namespace，并标记 ` istio-injection=enabled` 以应用自动注入 sidecar。
```
$ kubectl create namespace test-mesh
namespace/test-mesh created

$ kubectl label namespace test-mesh istio-injection=enabled
namespace/test-mesh labeled
```

部署 bookinfo 到这个 namespace。

```
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n test-mesh
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n test-mesh
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

从公网lb访问 http://your-lb-ip/productpage。不出意外，已经可以正常的展示官方的bookinfo示例页面了。

## 小结

我们完成了网格的安装，并可以通过公网访问到示例应用。下一步我们将实现监控，观测组件(grafana, kiali, tracing)的部署。
