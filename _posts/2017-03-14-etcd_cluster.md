---
tags: container
layout: post
title: etcd 集群部署方案
category: orchestration
---

etcd在整个Kubernetes集群中处于中心数据库的地位，<!--more-->为保证Kubernetes集群的高可用性，首先需要保证数据库不是单故障点。一方面，etcd需要以集群的方式进行部署，以实现etcd数据存储的冗余、备份与高可用性；另一方面，etcd存储的数据本身也应考虑使用可靠的存储设备。 

### 部署方案介绍

#### 第一种：[etcd operator](https://github.com/coreos/etcd-operator). 
比较新，处于beta阶段，简化了配置与管理，etcd server本身也是容器化部署，有快速scale能力。是基于k8s [`User Aggregated API Servers`](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/aggregated-api-servers.md)机制来实现的。<br/>
Requirements:
 - k8s 1.5.3+
 - etcd 3.0+

#### 第二种：传统方式
官方有[介绍文档](https://coreos.com/etcd/docs/latest/op-guide/clustering.html)，比较成熟，但灵活度不够，比如扩容性方面，节点依赖等。
 分三种情况:
  - Static：已知IP，逐个节点安装
  - etcd Discovery: 共享A discovery URL,须联网（依赖另一个K/V store）[本人推荐方式]
  - DNS Discovery;  需创建DNS记录


##### 1.static配置方式（要配置本方地址和其他人地址）

适用于在配置前已经明确各种信息的情况，比如集群的大小，各member的ip，端口等信息。

各member依靠配置得知其他member的联系地址。当然ip等信息可以通过环境变量传进去，不一定要写死。

 

假设需要建一个有三个节点的集群，三个节点地址分别为：10.0.1.10，10.0.1.11，10.0.1.12。

1.1、集群建立阶段

在第一个节点上执行：
``` sh
etcd --name etcd0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.10:2379 \
--initial-cluster-token etcd-cluster-2 \
--initial-cluster etcd0=http://10.0.1.10:2380,etcd1=http://10.0.1.11:2380,etcd2=http://10.0.1.12:2380 \
--initial-cluster-state new
```
在第二个节点上执行：
``` sh
etcd --name etcd1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.11:2379 \
--initial-cluster-token etcd-cluster-2 \
--initial-cluster etcd0=http://10.0.1.10:2380,etcd1=http://10.0.1.11:2380,etcd2=http://10.0.1.12:2380 \
--initial-cluster-state new
```
在第三个节点上执行：
``` sh
etcd --name etcd2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.12:2379 \
--initial-cluster-token etcd-cluster-2 \
--initial-cluster etcd0=http://10.0.1.10:2380,etcd1=http://10.0.1.11:2380,etcd2=http://10.0.1.12:2380 \
--initial-cluster-state new
```
 

1.2、运行阶段member异常恢复

假设一个节点etcd2异常重启，可以再执行如下命令拉起来

etcd --name etcd2  \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.12:2379 

static配置方式下，且处于运行阶段时，所有--initial-cluster参数没作用，带与不带都没有影响。


##### 2.discovery自发现方式（只配置本方地址，其他人地址从中介处获取）

依赖于第三方etcd服务。在“集群建立阶段”各member都向第三方etcd服务注册，也从其获取其他member的信息。就像有个中介一样。

在“集群运行阶段”，对第三方etcd不再有依赖。

 

以私有etcd方式地址做中介为例说明：

2.1、集群建立阶段

a、首先在中介处申请一块地方

有两种方式，私有和官网方式任选其一

    私有etcd方式：

    uuidgen 先生成一个标识78b12ad7-2c1d-40db-9416-3727baf686cb

    curl -X PUT http://192.168.1.163:20003/v2/keys/discovery/78b12ad7-2c1d-40db-9416-3727baf686cb/_config/size -d value=3

    官网方式：

    curl https://discovery.etcd.io/new?size=3
    返回 https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de

b、在各个节点上分别执行
```sh
etcd --name etcd0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.10:2379 \
-discovery http://192.168.1.163:20003/v2/keys/discovery/78b12ad7-2c1d-40db-9416-3727baf686cb \
--initial-cluster-state new
```
```sh
etcd --name etcd1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.11:2379 \
-discovery http://192.168.1.163:20003/v2/keys/discovery/78b12ad7-2c1d-40db-9416-3727baf686cb \
--initial-cluster-state new
```
```sh
etcd --name etcd2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.12:2379 \
-discovery http://192.168.1.163:20003/v2/keys/discovery/78b12ad7-2c1d-40db-9416-3727baf686cb \
--initial-cluster-state new
```
查看etcd集群的成员信息：`etcdctl member list`，用于后面配置API Server(参数etcd-servers)所需的信息

2.1、运行阶段member异常恢复

下面两条命令任选其一都可以起来。优选第一条
```sh
etcd --name etcd2 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.12:2379
```
 
```sh
etcd --name etcd2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
--listen-peer-urls http://0.0.0.0:2380 \
--listen-client-urls http://0.0.0.0:2379 \
--advertise-client-urls http://10.0.1.12:2379 \
-discovery http://192.168.1.163:20003/v2/keys/discovery/78b12ad7-2c1d-40db-9416-3727baf686cb \
--initial-cluster-state existing
```








使用方式,无须再配proxy
以kube-apiserver为例，将访问etcd集群的参数设置为：　--etcd-servers=http://10.0.0.1:4001,http://10.0.0.2:4001,http://10.0.0.3:4001 