# Introduction

Welcome to User Mode Linux

User Mode Linux is the first Open Source virtualization package (first release date 1991) and second virtualization package for an x86 PC.

It's going to be fun. 

## How is User Mode Linux Different from a Virtual Machine using Virtualization package X?

We have come to assume that virtualization also means some level of
hardware emulation. In fact, it does not. As long as a virtualization
package provides the OS with devices which the OS can recognize and
has a driver for, the devices do not need to emulate real hardware.
In fact, Most OSes today have built-in support for a number of "fake"
devices used only under virtualization.

User Mode Linux takes this concept to the ultimate extreme - there
is not a single real device in sight. It is 100% artificial or if
we use the correct term 100% paravirtual. All UML devices are abstract
concepts which map onto something provided by the host - files, sockets,
pipes, etc.

The other major difference between UML and various virtualization
packages is that there is a distinct difference between the way the UML 
kernel and the UML programs operate.

The UML kernel is just a process running on Linux - same as any other
program. It can be run by an unprivileged user and it does not require
anything in terms of special CPU features.

The UML userspace, however, is a bit different. The Linux kernel on the
host machine assists UML in intercepting everything the program running
on a UML instance is trying to do and making the UML kernel handle all
of its requests. 

This is different from other virtualization packages which do not make any
difference between the guest kernel and guest programs. This difference
results in a number of advantages and disadvantages of UML over let's say
QEMU which we will cover later in this document.

  
## Why Would I Want User Mode Linux?


1. If User Mode Linux kernel crashes, your host kernel is still fine. It
is not accelerated in any way (vhost, kvm, etc) and it is not trying to
access any devices directly.  It is, in fact, a process like any other.

1. You can run a usermode kernel as a non-root user (you may need to
arrange appropriate permissions for some devices).

1. You can run a very small VM with a minimal footprint for a specific 
task (f.e. 32M or less).

1. You can get extremely high performance for anything which is a "kernel
specific task" such as forwarding, firewalling, etc while still being 
isolated from the host kernel.

1. You can play with kernel concepts without breaking things.

1. You are not bound by "emulating" hardware, so you can try weird and
wonderful concepts which are very difficult to support when emulating real
hardware such as time travel and making your system clock dependent on
what UML does (very useful for things like tests). 

1. It's fun.

## Why not to run UML

1. The syscall interception technique used by UML makes it inherently slower
for any userspace applications. While it can do kernel tasks on par with most
other virtualization packages, its userspace is **slow**. The root cause is that
UML has a very high cost of creating new processes and threads (something
most Unix/Linux applications take for granted).

1. UML is strictly uniprocessor at present. If you want to run an application
which needs many CPUs to function, it is clearly the wrong choice.


