---
title: "记一次 Kubernetes 节点 OOM 的排查"
date: 2020-06-18T11:52:00+08:00
draft: false
tags: ["kubernetes", "OOM", "troubleshooting"]
slug: k8s-node-oom-troubleshooting
---

## 环境

- tke(腾讯云容器服务) 1.12.4 托管集群.
- 节点操作系统：ubuntu16.04.1 LTSx86_64，4.4.0-104-generic
- kube-proxy mode: ipvs
- Docker version 18.06.3-ce, build d7080c1

## 过程

集群内的业务rpc请求大量超时，与[kube-proxy的ipvs模式udp转发规则过期问题](/posts/k8s-ipvs-udp-rule-expire-5min/) 的现象一致。
从集群-事件中观察到以下事件，且该节点有coredns的pod。

```
2020-06-14 13:15:39  Warning Node 172.21.128.138.1618e2878157535e Rebooted  Node 172.21.128.138 has been rebooted, boot id: 281c2a66-2bb2-4274-b979-e1f50cc56fd
```
![47](/images/k8s-node-oom/node-reboot-event.png)

后客服反馈：CVM在当时发生了OOM。在得到云厂商客服的反馈后，运维同学没有继续跟进，只是简单的认为是编排中的资源配置问题，可能导致了OOM(我们的编排中, resources.limit.memory > resources.requests.memory)，对编排进行了调整。在后续的几天里，发生了4次类似的reboot情况，发生OOM时的时间点和内存监控图如下:

```
143, 2020-06-11 10:52
138, 2020-06-14 13:15
124, 2020-06-15 23:00
47, 2020-06-16 09:40
```

![47](/images/k8s-node-oom/47.png)
![124](/images/k8s-node-oom/124.png)
![138](/images/k8s-node-oom/138.png)
![143](/images/k8s-node-oom/143.png)


从以上的图表中发现：

- 发生OOM时，部分节点的free指标(绿色)是充裕的，按理不应该触发OOM。
- 发生OOM前，cache指标占80%。这可能是正常的。

我们将以上信息同步给了云厂商的技术支持，后反馈

```
ins-xxx
ins-yyy
在这两个节点上都有发现上面的内核报错。我反馈给相关同事进一步看下
```

找到内核报错的方式

```sh
# journalctl --since "2020-06-14 13:00" --until "2020-06-14 14:00"

...
Jun 14 13:14:38 host-172-21-128-138 kernel: ------------[ cut here ]------------
Jun 14 13:14:38 host-172-21-128-138 kernel: kernel BUG at /build/linux-SwhOyu/linux-4.4.0/include/linux/fs.h:2582!
-- Reboot --
Jun 14 13:15:06 host-172-21-128-138 systemd-journald[623]: Runtime journal (/run/log/journal/) is 8.0M, max 643.0M, 635.0M free.
Jun 14 13:15:06 host-172-21-128-138 kernel: Initializing cgroup subsys cpuset
Jun 14 13:15:06 host-172-21-128-138 kernel: Initializing cgroup subsys cpu
...
```

后要求提供 ins-xxx 这台服务器的 /sbin/auplink 文件

在一天后，云厂商技术支持反馈

```
CVM侧分析，ins-xxx这台服务器没有开启kdump，没有生成vmcore,无法进一步分析内核问题。建议您为节点开启下kdump，有故障产生vmcore，我们好进一步分析
```

打开kdump的方式见文档 [Ubuntu 开启 kdump](/files/ubuntu-enable-kdump.pdf)

### 节点资源预留策略

[为系统守护进程预留计算资源](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/) 描述了k8s的设计，以下是 kubelet 的启动参数(忽略了一些)，可以看到有配置相关的策略。

```sh
/usr/bin/kubelet --kube-reserved=cpu=140m,memory=3436Mi --eviction-hard=nodefs.available<10%,nodefs.inodesFree<5%,memory.available<100Mi --fail-swap-on=false
```
## 一些对策

- 开通所有节点的kdump，以便后续复现时，腾讯云侧能帮助排查。
- 调整资源配置。内存资源按request=limit的方式配置，从调度层面避免OOM的可能性，这会增加内存资源的需求，可能会需要增加节点或调整机型。
- 完善云监控中关于cvm, 容器服务相关项目的告警策略配置。
- 加强巡检。

## 总结

- 从目前收集的信息看，这次故障是内核bug导致的非预期的OOM，未定位原因。
- 业务超时的原因是dns解析超时导致。
- 出现故障时，没有第一时间进行更深层次的排查，失去一次获取真实原因的机会，且做出的决策可能是错误的，成本可能是巨大的。

## 后续

2020-06-23 18:35 又一节点出现了相同的情况，由于之前开启了kdump，在 /var/crash 下生成有文件，云厂商取走分析

06-24 14:44 反馈：“这个是促发了ubuntu的内核bug，导致的重启。看状态这个问题ubuntu还未发版本修复”。[ubuntu bug 1653498](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1653498)
![ubuntu-bug-1653498](/images/k8s-node-oom/ubuntu-bug-1653498.png)

07-01 11:13 反馈：“ 这个问题看下ubuntu的问题单目前ubuntu16.04这个问题还未有修复方法，因此目前也没法从代码层面确定其他发行版是否有该问题。从社区上报这个问题的基本都是ubuntu16.04版本，所以条件允许还是建议这边迁移到centos7“
![ubuntu-rebooted-bug](/images/k8s-node-oom/ubuntu-rebooted-bug.png)

以下是云厂商最终反馈

1. 这个内核bug在17年社区就有人提出来，但是一直没有修复
2. 而社区反馈的linux发行版都是ubuntu16
3. 无法从代码角度判断其他发行版是否也有问题，但是建议采用较新的内核版本，内核组同事给的建议是切到centos7
4. 最近一次问题跟之前报出来的问题（4个节点异常重启），属于同一个问题，我们这边建议咱切换
5. 同时建议切到tke定制化操作系统
