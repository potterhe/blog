---
title: kube-proxy的ipvs模式udp转发规则过期问题
date: 2019-12-03T23:59:59+08:00
draft: false
tags: ["kubernetes", "kube-proxy", "ipvs", "udp", "expire"]
---

我们在使用腾讯云容器服务(tke)的过程中，遭遇了kube-proxy的ipvs模式udp转发规则过期问题，过程记录。

## 环境

- tke 1.12.4 托管集群.
- 节点操作系统：ubuntu16.04.1 LTSx86_64
- kube-proxy ipvs模式 /usr/bin/kube-proxy --proxy-mode=ipvs --ipvs-min-sync-period=1s --ipvs-sync-period=5s --ipvs-scheduler=rr --masquerade-all=true --kubeconfig=/etc/kubernetes/kubeproxy-kubeconfig --hostname-override=172.21.128.111 --v=2
- 运行时：Docker version 18.06.3-ce, build d7080c1

## 操作和现象

1. 目标节点：172.21.128.109, 该节点有coredns, coredns的svc ClusterIP: 172.23.127.235。执行封锁后，执行drain操作。操作后的现象：业务大量报错“getaddrinfo failed: Name or service not known (10.010s)”,持续约8分钟.
2. 和腾讯云的伙伴复盘时关注dns的变化.操作后会看到新的pod在172.21.128.111节点生成，在集群的任意节点上查看ipvs规则,发现tcp规则已更新成新podip，但udp规则还是老的podip。

```sh
# kubectl drain 172.21.128.109

# kubectl get pod -n kube-system -o wide|grep dns
coredns-568cfc555b-4vdgk                1/1     Running   0          66s     172.23.3.41      172.21.128.111   <none>
coredns-568cfc555b-7zkfz                1/1     Running   0          77d     172.23.0.144     172.21.128.10    <none>

# ipvsadm -Ln|grep -A2 172.23.127.235:53
TCP  172.23.127.235:53 rr
  -> 172.23.0.144:53              Masq    1      0          0
  -> 172.23.3.41:53               Masq    1      0          0
UDP  172.23.127.235:53 rr
  -> 172.23.0.144:53              Masq    1      0          39229
  -> 172.23.2.15:53               Masq    0      0          39258
```

3. 腾讯云的伙伴反馈

```
ipvs udp转发规则的expire 时间为5分钟，如果不到超时时间的话，旧的转发规则链就还会在。进而导致新的udp请求走到旧的规则链（实际上后端rs已经没了）而失败。

这个社区在1.15解决
https://github.com/kubernetes/kubernetes/pull/77802
https://github.com/kubernetes/kubernetes/issues/76664

kube-proxy在清除旧的ipvs规则时，如果InActConn不等于0的话，就会不处理，直到等于0才处理。而上面的pr的节点办法是直接删除udp规则链，不需要等expirre这个优雅超时时间。
```

## 结论

ipvs的udp规则问题导致，所有的udp服务都有这个问题. 5分钟的服务不可用对于生产系统来说，不可接受。
