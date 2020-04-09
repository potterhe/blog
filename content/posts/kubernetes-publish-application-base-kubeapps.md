---
title: "基于 Kubeapps 的应用管理、发布系统"
date: 2020-03-14T11:52:00+08:00
draft: false
tags: ["kubernetes", "kubeapps", "chartmuseum", "发布系统"]
slug: a-kubeapps-based-application-deploying-and-managing-system
---

在引入 Kubernetes 时，我们需要提供对应的 CI/CD 方案。

## 需求&痛点

- 私有的统一认证，授权。角色，权限管控。
- 审计合规。主要包括测试人员发版，发版有历史，可审计追溯。
- 多云，多集群。
- 功能产品化。如：发版，扩容，回滚，下线。
- “回滚”机制要求“编排文件”也需要用版本控制工具管理起来，版本化。

## 选型

- rancher
- drone
- spinnaker
- helm 生态

## 为什么选择 Kubeapps

[Kubeapps](https://github.com/kubeapps/kubeapps)

    在整理这篇的时候，特意确认了 kubeapps 的最新架构，一些组件的名字已经发生了改变。

- 需求描述中的“编排文件版本化”，这正是 helm 的特性，虽然社区有其它的解决方案，但 helm 目前仍是那个应用最广泛的，成熟且有丰富的生态。
- kubeapps 基于 helm。
- kubeapps 是产品化的，而不是一个“轮子”。
- 封装良好的 api，极容易复用和扩展。这些 api 包括：整合了chartmuseum，并提供了chartsvc；tiller-proxy api；apprepository api；搜索 api，基于mongo的缓存，定时同步。
- 权限透明。它的权限完全是基于 Kubernetes 的 RBAC 机制，是透明的。
- 前端 React。

## chartmuseum

[chartmuseum](https://github.com/helm/chartmuseum) 是 helm 的 charts 仓库。

- 无状态
- 几乎支持所有云的对象存储作为后端，简单，安全。

由于我们的业务主要是在腾讯云，我们希望用其 cos 服务作为存储 backend。我们为社区提供了两个pr，完善了 chartmuseum 对腾讯云 cos 的支持。

- https://github.com/helm/chartmuseum/pull/239
- https://github.com/chartmuseum/storage/pull/30

## 系统架构

![mykubeapps](/images/mykubeapps.svg)


### charts 的维护

charts 通过 git 进行版本控制，通过 MergeRequest(gitlab 的概念，github 叫 PullRequest) 提交变更，合并操作触发 CI 自动推送到 chartmuseum。下面是 CI 脚本，helm 的 push 插件需要自行安装。

```sh
#!/bin/bash
set -e

charts=$(git diff --name-only HEAD^ HEAD|awk -F/ '{print $1}'|sort|uniq)
for chart in $charts;
do
    echo $chart
    if [ -d "$chart" ]; then
        helm push -f $chart $CHARTMUSEUM_URL --username=$CHARTMUSEUM_BASIC_AUTH_USER --password=$CHARTMUSEUM_BASIC_AUTH_PASS
    fi
done
```
