---
tags:
  - linux
  - 'virtualization'
layout: post
title: 'Linux硬件虚拟化的几个级别'
category: linux
---
# 虚拟化的几个级别，及KVM的一点笔记

<!--more-->

Full virtualization (全虚拟化)，它是指所有硬件（CPU/内存/硬盘/网卡等）都被虚拟出来，由一个Hypervisor(就是一VMM)统一管理交互，特点就是GUEST OS可以不做任何修改，完全透明的运行。


Paravirtualization(部分虚拟化)，它是并非所有硬件都被虚拟，相应的，它提供一个特定的API,去修改GUEST OS而使用这些API，从而消除一些瓶颈，提升性能。但随着INTEL/AMD提供了硬件级别的虚拟方案，全虚拟化的性能问题已经较好的解决了。


OS-Virtualizaiton(操作系统级别虚拟化)，这是目前发展较快的一个领域，它提供多个隔离的运行环境，但这些运行环境却共享一个OS Kernel，典型的例子就是Docker。





KVM是全虚拟化方案的一种，利用CPU的硬件(Intel VT or AMD-V)支持 ,它对外提供一个/dev/kvm接口。
见下图
![](/assets/kvm/kvm_arch_map.jpg)


# CPU虚拟化

X86体系结构CPU虚拟化技术的称为 Intel VT-x 技术，引入了VMX，提供了两种处理器的工作环境。 VMCS 结构实现两种环境之间的切换。 VM Entry 使虚拟机进去guest模式，VM Exit 使虚拟机退出guest模式。

VMM调度guest执行时，qemu 通过 ioctl 系统调用进入内核模式，在 KVM Driver中获得当前物理 CPU的引用。之后将guest状态从VMCS中读出， 并装入物理CPU中。执行 VMLAUCH 指令使得物理处理器进入非根操作环境，运行guest OS代码。

当 guest OS 执行一些特权指令或者外部事件时， 比如I/O访问，对控制寄存器的操作，MSR的读写等， 都会导致物理CPU发生 VMExit， 停止运行 Guest OS，将 Guest OS保存到VMCS中， Host 状态装入物理处理器中， 处理器进入根操作环境，KVM取得控制权，通过读取 VMCS 中 VM_EXIT_REASON 字段得到引起 VM Exit 的原因。 从而调用kvm_exit_handler 处理函数。 如果由于 I/O 获得信号到达，则退出到userspace模式的 Qemu 处理。处理完毕后，重新进入guest模式运行虚拟 CPU。

# 内存虚拟化

OS对于物理内存主要有两点认识：1.物理地址从0开始；2.内存地址是连续的。VMM接管了所有内存，但guest OS的对内存的使用就存在这两点冲突了，除此之外，一个guest对内存的操作很有可能影响到另外一个guest乃至host的运行。VMM的内存虚拟化就要解决这些问题。

在OS代码中，应用也是占用所有的逻辑地址，同时不影响其他应用的关键点在于有线性地址这个中间层；解决方法则是添加了一个中间层：guest物理地址空间；guest看到是从0开始的guest物理地址空间（类比从0开始的线性地址），而且是连续的，虽然有些地址没有映射；同时guest物理地址映射到不同的host逻辑地址，如此保证了VM之间的安全性要求。

这样MEM虚拟化就是GVA->GPA->HPA的寻址过程，传统软件方法有影子页表，硬件虚拟化提供了EPT支持。


kvm的源码还不小，可以[这里](http://git.kernel.org/cgit/virt/kvm/kvm.git)查看









参考[https://en.wikipedia.org/wiki/Hardware_virtualization](https://en.wikipedia.org/wiki/Hardware_virtualization)


---
