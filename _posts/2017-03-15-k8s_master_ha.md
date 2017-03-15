---
tags: container
layout: post
title: k8s master HA 高可用方案
category: orchestration
---

### 参考资料

- HA的几种实现方式： https://www.ibm.com/developerworks/community/blogs/RohitShetty/entry/high_availability_cold_warm_hot?lang=en

- k8s中ha的目前实现状态： https://github.com/kubernetes/kubernetes/blob/master/docs/admin/high-availability.md

- k8s中HA的design proposal: https://github.com/kubernetes/kubernetes/blob/master/docs/proposals/high-availability.md 



这里其实是指Master节点的HA方式，官方目前采取的方式： 利用etcd实现master 选举，从多个Master中得到一个kube-apiserver 保证至少有一个master可用，实现high availability。对外以loadbalancer的方式提供入口。这种方式可以用作ha，但仍未成熟，据了解，未来会更新升级ha的功能。


#### 最终呈现效果图：

![效果图](https://kubernetes.io/images/docs/ha.svg =200)

这种方式是基于容器部署的，把api-server,scheduler,controller-manager均放于容器内。

## 配置步骤

假定三个master节点的ip分别是10.0.1.10，10.0.1.11，10.0.1.12, proxy(load balancer) ip为10.0.1.13

### 第一步：给每台master安装kubelet，以便于后续容器化安装各个组件。
下载[kubelet binary](https://storage.googleapis.com/kubernetes-release/release/v0.19.3/bin/linux/amd64/kubelet),安装
`systemctl enable kubelet` and `systemctl enable docker`

### 第二步：建etcd集群，参考前篇()；

### 第三步：配置master上的api-server组件，
Installing configuration files
1.`touch /var/log/kube-apiserver.log`
2.create a `/srv/kubernetes/` directory on each node
```sh
This directory includes:
    basic_auth.csv - basic auth user and password
    ca.crt - Certificate Authority cert
    known_tokens.csv - tokens that entities (e.g. the kubelet) can use to talk to the apiserver
    kubecfg.crt - Client certificate, public key
    kubecfg.key - Client certificate, private key
    server.cert - Server certificate, public key
    server.key - Server certificate, private key
```
最容易的方式就是从一个已经运行的master里拷贝出这些文件。
Starting the API Server
以容器的方式起，Pods文件请[下载](https://kubernetes.io/docs/admin/high-availability/kube-apiserver.yaml)，之后放在`/etc/kubernetes/manifests/`,kubelet会自动监视这个目录的变化，并启动对应Pods.

### 第四步：给API servers加Proxy [load balancing] 

因为现在有了三个API server在监听10.0.1.10:8080,10.0.1.11:8080,10.0.1.12:8080，可用nginx做转发，
```sh
upstream backend {
             #ip_hash;
             server 10.0.1.10:8080;
             server 10.0.1.11:8080;
             server 10.0.1.12:8080;
         }

server {
        listen       8080;
        server_name 10.0.1.13:8080;
        location / {
             #反向代理的地址
             proxy_pass http://backend;  
        }
}
```
### 第五步：在每个master节点Starting scheduler，关键点：开启--leader-elect=true

先建log文件，防止Docker把它mount成目录:`touch /var/log/kube-scheduler.log`
以容器的方式起，Pods文件请[下载](https://kubernetes.io/docs/admin/high-availability/kube-scheduler.yaml)，注意修改master url为proxy ip.之后放在`/etc/kubernetes/manifests/`,kubelet会自动监视这个目录的变化，并启动对应Pods.

### 第六步：在每个master节点Starting controller-manager，关键点：开启--leader-elect=true

先建log文件，防止Docker把它mount成目录:`touch /var/log/kube-controller-manager.log`
以容器的方式起，Pods文件请[下载](https://kubernetes.io/docs/admin/high-availability/kube-controller-manager.yaml)，注意修改master url为proxy ip,之后放在`/etc/kubernetes/manifests/`,kubelet会自动监视这个目录的变化，并启动对应Pods.



## 参考资料
