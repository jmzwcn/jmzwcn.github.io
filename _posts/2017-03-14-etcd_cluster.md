---
tags: container
layout: post
title: etcd 集群部署方案
category: orchestration
---

etcd在整个Kubernetes集群中处于中心数据库的地位，<!--more-->为保证Kubernetes集群的高可用性，首先需要保证数据库不是单故障点。一方面，etcd需要以集群的方式进行部署，以实现etcd数据存储的冗余、备份与高可用性；另一方面，etcd存储的数据本身也应考虑使用可靠的存储设备。 

## 部署方案介绍

### 第一种：[etcd operator](https://github.com/coreos/etcd-operator). 
为k8s量身定做，比较新，处于beta阶段，简化了配置与管理，etcd server本身也是容器化部署，有快速scale能力。是基于k8s [`User Aggregated API Servers`](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/aggregated-api-servers.md)机制来实现的。<br/>
Requirements:
 - k8s 1.5.3+
 - etcd 3.0+

### 第二种：传统方式
官方有[介绍文档](https://coreos.com/etcd/docs/latest/op-guide/clustering.html)，比较成熟，但灵活度不够，比如扩容性方面，节点依赖等。
 分三种情况:
  - 1.Static：已知集群所有节点IP，逐个节点安装
  - 2.etcd Discovery: 共享一个 discovery URL，[本人推荐方式]<br/>
  优点：和静态的比，不用提前知道集群所有节点的IP信息；<br/>
  缺点：须联网，不然须先建一个单节点etcd,依赖已经存在的K/V服务。
  - 3.DNS Discovery:  基于DNS SVR记录<br/>
  优点：节点替换方便，比如API server里的etcd-servers参数用域名的话，不用动；<br/>
  缺点：节点增加/删除的话和2比，会多一个增加/删除DNS记录的操作。


#### 1.static配置方式（要配置本节点地址和其他节点地址）

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


#### 2.discovery自发现方式（只配置本节点地址，其他节点地址从中介[discoveryURL]处获取）

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
注意：如果集群已经达到初始size的话，再继续添加的话，这节点会变成proxy，除非添加之前先给集群扩容`etcdctl member add ***`

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

#### 3.DNS SRV记录方式
首先须再DNS SRV里添加如下记录:
```sh
$ dig +noall +answer SRV _etcd-server._tcp.example.com
_etcd-server._tcp.example.com. 300 IN   SRV 0 0 2380 infra0.example.com.
_etcd-server._tcp.example.com. 300 IN   SRV 0 0 2380 infra1.example.com.
_etcd-server._tcp.example.com. 300 IN   SRV 0 0 2380 infra2.example.com.
```
再添加A记录，效果如下：
```sh
$ dig +noall +answer infra0.example.com infra1.example.com infra2.example.com
infra0.example.com. 300 IN  A   10.0.1.10
infra1.example.com. 300 IN  A   10.0.1.11
infra2.example.com. 300 IN  A   10.0.1.12
```
做好了上述两步DNS的配置，就可以使用DNS启动etcd集群了。配置DNS解析的url参数为-discovery-srv，其中某一个节点地启动命令如下
```sh
$ etcd -name infra0 \
-discovery-srv example.com \
-initial-advertise-peer-urls http://infra0.example.com:2380 \
-initial-cluster-token etcd-cluster-1 \
-initial-cluster-state new \
-advertise-client-urls http://infra0.example.com:2379 \
-listen-client-urls http://infra0.example.com:2379 \
-listen-peer-urls http://infra0.example.com:2380
```
当然，你也可以直接把节点的域名改成IP来启动。


### 添加/更新/删除etcd节点 [注意操作后须在k8s API server和flannel里修改对应etcd-servers/etcd-endpoints参数]

- 添加：
要遵循先这册（在当前集群任意节点上执行`etcdctl member add infra3 http://10.0.1.13:2380`）再启动的原则，并且注意-initial-cluster-state参数为existing，若为new的话，启动的节点会一直为proxy（不参与选举，但可正常转发读写），即使某台member节点挂掉。

```sh
$ etcdctl member add infra3 http://10.0.1.13:2380
added member 9bf1b35fc7761a23 to cluster

ETCD_NAME="infra3"
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
ETCD_INITIAL_CLUSTER_STATE=existing

etcdctl 在注册完新节点后，会返回一段提示，包含3个环境变量。然后在第二部启动新节点的时候，带上这3个环境变量即可。

$ export ETCD_NAME="infra3"
$ export ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
$ export ETCD_INITIAL_CLUSTER_STATE=existing
$ etcd -listen-client-urls http://10.0.1.13:2379 -advertise-client-urls http://10.0.1.13:2379  -listen-peer-urls http://10.0.1.13:2380 -initial-advertise-peer-urls http://10.0.1.13:2380
```

- 更新
如果你想更新一个节点的 IP(peerURLS)，首先你需要知道那个节点的 ID。你可以列出所有节点，找出对应节点的 ID。
```sh
$ etcdctl member list
6e3bd23ae5f1eae0: name=node2 peerURLs=http://localhost:23802 clientURLs=http://127.0.0.1:23792
924e2e83e93f2560: name=node3 peerURLs=http://localhost:23803 clientURLs=http://127.0.0.1:23793
a8266ecf031671f3: name=node1 peerURLs=http://localhost:23801 clientURLs=http://127.0.0.1:23791

在本例中，我们假设要更新 ID 为 a8266ecf031671f3 的节点的 peerURLs 为：http://10.0.1.10:2380

$ etcdctl member update a8266ecf031671f3 http://10.0.1.10:2380
Updated member with ID a8266ecf031671f3 in cluster
```

- 删除：
```sh
假设我们要删除 ID 为 a8266ecf031671f3 的节点

$ etcdctl member remove a8266ecf031671f3
Removed member a8266ecf031671f3 from cluster

执行完后，目标节点会自动停止服务，并且打印一行日志：

etcd: this member has been permanently removed from the cluster. Exiting.

如果删除的是 leader 节点，则需要耗费额外的时间重新选举 leader。
```


### 使用方式
无须再配proxy,访问etcd集群的参数设置:<br/>

kube-apiserver参数：--etcd-servers=http://10.0.0.1:4001,http://10.0.0.2:4001,http://10.0.0.3:4001 <br/>
flannel参数：--etcd-endpoints=http://127.0.0.1:4001: a comma-delimited list of etcd endpoints.
<br/>
<br/>
### urls内部负载均衡
- API Server etcd-servers参数的load balancer实现<br/>
引用etcd的simpleBalancer[https://github.com/coreos/etcd/blob/master/clientv3/balancer.go]<br/>
算法就是同时去connect配的这些地址，谁先返回，就把它作为一个pinned address,之后就一直用它，除非它有异常；<br/>

其实它是把gRPC的默认实现roundRobin[https://github.com/grpc/grpc-go/blob/master/balancer.go#L156]重写了。<br/><br/>


- 健康检查<br/>
实现是lbWatcher(基于transportMonitor) <br/>
https://github.com/grpc/grpc-go/blob/d50cf2db166eaff3f2429425758d12205085eb5b/clientconn.go#L856<br/><br/>

flannel的etcd-endpoints参数与API Server一样，也是调的etcd的实现。<br/><br/>

- 更多参考链接<br/>
https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/storage/storagebackend/factory/etcd3.go#L45<br/>
https://github.com/coreos/etcd/blob/master/clientv3/config.go#L27<br/>
https://github.com/coreos/etcd/blob/master/clientv3/client.go#L351<br/>


[Fore more](https://coreos.com/etcd/docs/latest/op-guide/clustering.html#etcd-discovery)