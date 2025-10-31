# A OpenBSD VMM accelerator back-end for QEMU
> Saeed Mahjoob, Shamsipour technical and vocational college
> `saeed@cloud9p.org`

## Abstract
We propose a new accelerator back-end for QEMU^[[Quick EMUlator: A Portable computer emulator](https://qemu.org)]
for OpenBSD's VMM hypervisor^[[Virtual Machine Monitor](https://cvsweb.openbsd.org/src/sys/dev/vmm/)],
which aims to improve virtualization framework in BSD systems and enable more workflows
and use cases, specifically those related to isolation, sand-boxing and virtualization. 


## Introduction
In last few decades, virtualization have become one of the most important aspects of modern
computing, from servers, to desktops and embedded solutions for isolation capabilities,
ease of use and security gains attained by the hypervisors and virtual machines.

OpenBSD operating system , while fairly well known for it's security oriented
software stack (OpenSSH, `doas`, LibreSSL and pf firewall to name a few) have historically
been a bit behind in the virtualization until the arrival of [vmm(4)](https://openbsd.org/vmm) driver and [vmd(8)](https://man.openbsd.org/vmd)
tool-set in 2016. while this stack works fairly well for what it promises to offer,
it only focuses on few specific workflows, for example it only offers
virtio^[[A standard for (para-)virtualized devices](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html)]
devices for Ethernet and disk controllers,
which do not work on older guests without modifications.
Controlling virtual machines is only possible via serial console, although it is easier
to both implement and use, requires configuration in guests and disallows usage of
operating systems that operate in graphical interface, such as Android and Windows.

The current model of virtualization in OpenBSD is depicted in Figure 1.

![Current VMM stack](vmd.svg)

QEMU, on the other hand is very portable, can emulate vast amount of devices
and supports decent range of workflows. However, the overhead of software
emulation is very high and the default acceleration back-end which is called
TCG^[[Tiny Code Generator, a software based accelerator](https://www.qemu.org/docs/master/devel/index-tcg.html)]
has inadequate performance for modern software.

The standard QEMU solution for this is to implement hardware assisted accelerator,
which talks to a hypervisor such as Linux's KVM^[[Kernel Virtual Machine](https://linux-kvm.org)] or Xen^[https://xenproject.org].

While it might have been possible to port KVM or Xen to OpenBSD, it would've required
tremendous effort, as hypervisors are very complex, highly tied to both software environment
and hardware, and can cause kernel panics as they include kernel level code.
Thus reusing existing infrastructure would get us to the goal with less effort and better results.

The solution we propose is to implement an back-end which talks to `/dev/vmm` using ioctl(2) system call,
and handles management of vCPUs^[Virtual CPUs], without the overhead of emulation that TCG currently suffers from, as shown in Figure 2.

![Proposed VMM stack](qemu.svg)
