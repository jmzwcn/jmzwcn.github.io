---
tags: container
layout: post
title: k8s master HA 高可用方案
category: orchestration
---





主要是指Master节点的 HA方式 官方推荐 利用etcd实现master 选举，从多个Master中得到一个kube-apiserver 保证至少有一个master可用，实现high availability。对外以loadbalancer的方式提供入口。这种方式可以用作ha，但仍未成熟，据了解，未来会更新升级ha的功能。

![效果图](https://kubernetes.io/images/docs/ha.svg)

这种方式是基于容器部署的，把api-server,scheduler,controller-manager均放于容器内。

配置方式


第一步：给每台master安装kubernete，以便于后续容器化安装各个组件。
下载[kubelet binary](https://storage.googleapis.com/kubernetes-release/release/v0.19.3/bin/linux/amd64/kubelet),安装
`systemctl enable kubelet` and `systemctl enable docker`

第二步：建etcd集群，参考前篇()；

配置master上的api-server组件，

TODO

参考资料

HA的几种实现方式： https://www.ibm.com/developerworks/community/blogs/RohitShetty/entry/high_availability_cold_warm_hot?lang=en

k8s中ha的目前实现状态： https://github.com/kubernetes/kubernetes/blob/master/docs/admin/high-availability.md

k8s中HA的design proposal: https://github.com/kubernetes/kubernetes/blob/master/docs/proposals/high-availability.md 