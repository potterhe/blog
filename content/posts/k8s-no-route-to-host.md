---
title: Kubernetes "no route to host"问题
date: 2019-12-13T23:59:59+08:00
draft: false
tags: ["kube-proxy", "ipvs", "no route to host"]
---

我们在使用腾讯云容器服务(tke)的过程中，遇到"no route to host"问题，这里记录为运维日志。

## 环境

- tke 1.12.4(1.14.3) 托管集群.
- 节点操作系统：ubuntu16.04.1 LTSx86_64
- kube-proxy ipvs模式 /usr/bin/kube-proxy --proxy-mode=ipvs --ipvs-min-sync-period=1s --ipvs-sync-period=5s --ipvs-scheduler=rr --masquerade-all=true --kubeconfig=/etc/kubernetes/kubeproxy-kubeconfig --hostname-override=172.21.128.111 --v=2
- 运行时：Docker version 18.06.3-ce, build d7080c1

## 排查过程

请详见tke团队roc的文章[Kubernetes 疑难杂症排查分享: 诡异的 No route to host](https://imroc.io/posts/kubernetes/no-route-to-host/)

## 其它找到的一些有用的文章

- https://engineering.dollarshaveclub.com/kubernetes-fixing-delayed-service-endpoint-updates-fd4d0a31852c
- https://fuckcloudnative.io/posts/kubernetes-fixing-delayed-service-endpoint-updates/ 中文版本
- 五元组：源IP地址，源端口，目的IP地址，目的端口，和传输层协议这五个量组成的一个集合
