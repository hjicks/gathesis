# A OpenBSD VMM Accelerator Back-end for QEMU

*Third draft*

> Saeed Mahjoob, Shamsipour technical and vocational college
> `saeed@cloud9p.org`

## Abstract
QEMU^[[Quick EMUlator: A Portable machine emulator](https://qemu.org)]
is the de-facto virtualization tool in open-source operating systems,
due to it's light weight, flexibility and portability. It's performance however
is **very** slow without hardware-assisted accelerators.
We propose a new accelerator back-end for QEMU
for OpenBSD's VMM hypervisor^[[Virtual Machine Monitor](https://cvsweb.openbsd.org/src/sys/dev/vmm/)],
with goal of better performance than the
TCG^[[Tiny Code Generator, a software based accelerator](https://www.qemu.org/docs/master/devel/index-tcg.html)]
accelerator, which is the current default on OpenBSD.

## Introduction
In last few decades, virtualization has become one of the most important aspects of modern
computing, from servers, to desktops and embedded solutions for isolation capabilities,
ease of use and security gains attained by the hypervisors and virtual machines.

OpenBSD operating system, while fairly well known for its security oriented
software stack (OpenSSH, `doas`, LibreSSL and the pf firewall to name a few) have historically
been behind in the virtualization until the arrival of
[*vmm*(4)](https://openbsd.org/vmm) driver and [*vmd*(8)](https://man.openbsd.org/vmd)
daemon in 2016. While this stack works fairly well for what it promises to offer,
it only focuses on few specific workflows. For example it only offers
virtio^[[A standard for (para-)virtualized devices](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html)]
devices for Ethernet and disk controllers, which does not work on older guests without modifications.
Controlling virtual machines is only possible via serial console, although it is easier
to both implement and use, requires configuration in guests and disallows usage of
operating systems that operate in graphical interface, such as Android and Windows.

The current model of virtualization in OpenBSD is depicted in Figure 1.

![Current VMM stack](vmd.svg)

QEMU, on the other hand can emulate vast amount of devices and supports decent range of workflows.
However, the overhead of software emulation is very high and
the default acceleration back-end which is called has inadequate performance for modern software.

The standard QEMU solution for this is to implement hardware assisted accelerator,
which talks to a hypervisor such as Linux's KVM^[[Kernel Virtual Machine](https://linux-kvm.org)]
or Xen^[https://xenproject.org].

While it might have been possible to port KVM or Xen to OpenBSD, it would have required
tremendous effort, as the hypervisors are generally very complex, highly tied to both software environment
and hardware, and can cause kernel panics since they include kernel level code.
Thus reusing existing infrastructure would get us to the goal with less effort and better results.

The solution we propose is to implement a back-end which talks to
`/dev/vmm` using *ioctl*(2) system call,
and handles management of vCPUs^[Virtual CPUs], without the overhead of emulation
that TCG currently suffers from, as shown in Figure 2.

![Proposed VMM stack](qemu.svg)

## Implementation
There are two major changes that must be done to currently available software:

* QEMU accelerator
* VMM changes

QEMU changes mostly consist of add C wrappers for existing interfaces, adapting them
to interfaces that already exists in QEMU, all of which are user-space code, dealing with
creation, initialization, halting, and destroying a vCPU,
as well as handling the event loop that does the input/output of the virtual machine.

On the other hand VMM changes are mostly made out of enhancements to current infrastructure
of OpenBSD, such as adding features that makes VMM behave more like KVM or NVMM, fixing bugs,
and other misc changes. Unlike Changes on QEMU, these are made out of kernel-space code,
which is less trivial and more prone to difficult to debug bugs.

### Inside of a hypervisor
Inner workings of hypervisors, are mostly similar, while details do differ.
Figure 3 shows a simplified version of virtual machine life cycle.

![Life cycle of a virtual machine](vmlifecycle.svg)

At the request of user, emulator starts the virtual machine and sets up various peripherals,
memory, bus and other hardware pieces you may think of, including CPU. But unlike the usual set,
CPUs are not emulated^[Unless it is, but we are talking about hardware assisted virtualization],
thus, it needs to ask the hardware itself to do the work. In doing so, we switch from
emulator which runs in user-space, to the guest vCPU, executing instructions until some sort of special interrupts happen.
In Intel terminology those interrupts are known as *VMExit*s, which can handle I/O requests, NMIs^[Non mask-able interrupts],
requests to pause or shutdown the virtual machine and many other.

Once a *VMExit* is issued, it is up to the hypervisor to handle it, as its not of guest's concern
nor the emulator. Hypervisor can then propagate the interrupt to the emulator, handle it on it's
own^[Overhead of context-switches sometimes makes this approach worth the trouble, FreeBSD does this for example for some devices],
halt the (guest) machine (to debug it) or ignore it and resume the virtual machine.

### QEMU Accelerator
QEMU already ships with several accelerators, some of which are:

* KVM
  * Linux, many platforms
* WHPX
  * Windows, x86
* HVF
  * MacOS, 64-bit x86 and arm
* NVMM
  * NetBSD, DrangonflyBSD, x86
* TCG
  * Many operating systems and many platforms

For example, if compiled with NVMM accelerator:
```
$ qemu-system-x86_64 --accel=nvmm <...>
```

Would enable that accelerator. Each accelerator controls the state of vCPU, which is defined as
a pointer-to-struct shared between the hardware emulator and CPU accelerators, called `CPUState`.
```
struct CPUState {
    int thread_id;
    bool created, running, has_waiter;
    bool stop, stopped;

    AddressSpace *as;
    MemoryRegion *memory;

    /* Use by accel-block: CPU is executing an ioctl() */
    QemuLockCnt in_ioctl_lock;

    bool vcpu_dirty;
    AccelCPUState *accel;
	
	/* snip... */
};
```

Each accelerator implements two classes, `AccelClass` and `AccelOpsClass`
## Related work and sources
Sources of this document is available at ![Github](https://github.com/hjicks/gathesis).

### VMM

- https://www.openbsd.org/papers/asiabsdcon2017-vmm-slides.pdf
- https://www.openbsd.org/papers/asiabsdcon2016-vmm-slides.pdf
- https://www.openbsd.org/papers/asiabsdcon2016-vmd-slides.pdf

### NVMM

- https://blog.netbsd.org/tnf/entry/from_zero_to_nvmm
- https://netbsd.org/~kamil/bhyvecon_tokyo2019.html

### QEMU
- https://dumrich.github.io/GSoC25-Blog/posts/qemu-accel/
