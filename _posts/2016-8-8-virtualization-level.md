---
tags:
  - linux
  - 'virtualization'
layout: post
title: 'Linux硬件虚拟化的几个级别'
category: linux
---
虚拟化的几个级别，及KVM的一点笔记

<!--more-->

Full virtualization (全虚拟化)，它是指所有硬件（CPU/内存/硬盘/网卡等）都被虚拟出来，由一个Hypervisor(就是一VMM)统一管理交互，特点就是GUEST OS可以不做任何修改，完全透明的运行。


Paravirtualization(部分虚拟化)，它是并非所有硬件都被虚拟，相应的，它提供一个特定的API,去修改GUEST OS而使用这些API，从而消除一些瓶颈，提升性能。但随着INTEL/AMD提供了硬件级别的虚拟方案，全虚拟化的性能问题已经较好的解决了。


OS-Virtualizaiton(操作系统级别虚拟化)，这是目前发展较快的一个领域，它提供多个隔离的运行环境，但这些运行环境却共享一个OS Kernel，典型的例子就是Docker。





KVM是全虚拟化方案的一种，利用CPU的硬件(Intel VT or AMD-V)支持 ,它对外提供一个/dev/kvm接口。
见下图
![](/assets/kvm/kvm-arch.jpg)

kvm的源码还不小，可以[这里](http://git.kernel.org/cgit/virt/kvm/kvm.git)查看









参考[https://en.wikipedia.org/wiki/Hardware_virtualization](https://en.wikipedia.org/wiki/Hardware_virtualization)


---
