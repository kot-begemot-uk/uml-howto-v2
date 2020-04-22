# Contributing to UML and Developing with UML

UML is an excellent platform to develop new Linux kernel concepts -
filesystems, devices, virtualization, etc. It provides unrivalled opportunities to create and
test them without being constrained to emulating specific hardware.

Example - want to try how linux will work with 4096 "proper" network devices?
Not an issue with UML. At the same time, this is something which is difficult
other virtualization packages - they are constrained by the number of devices allowed
on the hardware bus they are trying to emulate (f.e. 16 on a PCI bus in qemu).

If you have something to contribute such as a patch, a bugfix, a new feature, please
send it to linux-um@lists.infradead.org

Please follow all standard Linux patch guidelines such as cc-ing relevant maintainers
and run ./sripts/check-patch.pl on your patch.

Note - the list does not accept HTML or attachments, all emails must be formatted as plain
text.

Developing always goes hand in hand with debugging. First of all, you can always run
UML under gdb and there will be a whole section later on on how to do that. That, however,
is not the only way to debug a linux kernel. Quite often adding tracing statements and/or
using UML specific approaches such as ptracing the UML kernel process are significantly
more informative.

## Tracing UML

When running UML consists of a main kernel thread and a number of helper threads. The ones
of interest for tracing are NOT the ones that are already ptraced by UML as a part of
its MMU emulation. 

These are usually the first three threads visible in a ps display. The one with the lowest
PID number and using most CPU is usually the kernel thread. The other threads are the disk
(ubd) device helper thread and the sigio helper thread. 

running ptrace on this thread usually results in the following picture:
```
host$ strace -p 16566
--- SIGIO {si_signo=SIGIO, si_code=POLL_IN, si_band=65} ---
epoll_wait(4, [{EPOLLIN, {u32=3721159424, u64=3721159424}}], 64, 0) = 1
epoll_wait(4, [], 64, 0)                = 0
rt_sigreturn({mask=[PIPE]})             = 16967
ptrace(PTRACE_GETREGS, 16967, NULL, 0xd5f34f38) = 0
ptrace(PTRACE_GETREGSET, 16967, NT_X86_XSTATE, [{iov_base=0xd5f35010, iov_len=832}]) = 0
ptrace(PTRACE_GETSIGINFO, 16967, NULL, {si_signo=SIGTRAP, si_code=0x85, si_pid=16967, si_uid=0}) = 0
ptrace(PTRACE_SETREGS, 16967, NULL, 0xd5f34f38) = 0
ptrace(PTRACE_SETREGSET, 16967, NT_X86_XSTATE, [{iov_base=0xd5f35010, iov_len=2696}]) = 0
ptrace(PTRACE_SYSEMU, 16967, NULL, 0)   = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_TRAPPED, si_pid=16967, si_uid=0, si_status=SIGTRAP, si_utime=65, si_stime=89} ---
wait4(16967, [{WIFSTOPPED(s) && WSTOPSIG(s) == SIGTRAP | 0x80}], WSTOPPED|__WALL, NULL) = 16967
ptrace(PTRACE_GETREGS, 16967, NULL, 0xd5f34f38) = 0
ptrace(PTRACE_GETREGSET, 16967, NT_X86_XSTATE, [{iov_base=0xd5f35010, iov_len=832}]) = 0
ptrace(PTRACE_GETSIGINFO, 16967, NULL, {si_signo=SIGTRAP, si_code=0x85, si_pid=16967, si_uid=0}) = 0
timer_settime(0, 0, {it_interval={tv_sec=0, tv_nsec=0}, it_value={tv_sec=0, tv_nsec=2830912}}, NULL) = 0
getpid()                                = 16566
clock_nanosleep(CLOCK_MONOTONIC, 0, {tv_sec=1, tv_nsec=0}, NULL) = ? ERESTART_RESTARTBLOCK (Interrupted by signal)
--- SIGALRM {si_signo=SIGALRM, si_code=SI_TIMER, si_timerid=0, si_overrun=0, si_value={int=1631716592, ptr=0x614204f0}} ---
rt_sigreturn({mask=[PIPE]})             = -1 EINTR (Interrupted system call)

```

This is a typical picture from a mostly idle UML instance

1. UML interrupt controller uses epoll - this is UML waiting for IO interrupts:
`epoll_wait(4, [{EPOLLIN, {u32=3721159424, u64=3721159424}}], 64, 0) = 1`
1. The sequence of ptrace calls is part of MMU emulation and runnin the UML userspace
1. `timer_settime` is part of the UML high res timer subsystem mapping timer requests from inside
UML onto the host high resultion timers.
1. `clock_nanosleep` is UML going into idle (similar to the way a PC will execute an ACPI idle).

As you can see UML will generate quite a bit of output even in idle. The output can be very informative when observing IO. It shows
the actual IO calls, their arguments and returns values.

## Kernel debugging

You an run UML under gdb now, though it will not necessarily agree to be started under it. If 
you are trying to track a runtime bug, it is much better to attach gdb to a running UML instance
and let UML run.

Assuming the same PID number as in the previous example, this would be:
```
gdp -p 16566
```
This will STOP the UML instance, so you must enter `cont` at the GDB command line to request it to
continue.

# Developing Device Drivers

Nearly all UML drivers are monolithic.
While it is possible to build a UML driver as a kernel module, that
limits the possible functionality to in-kernel only and non-UML specific.

The reason for this is that in order to really leverage UML, one needs to write a
piece of userspace code which maps driver concepts onto actual userspace host calls.
This forms the so called "user" portion of the driver. While it can reuse a lot of
kernel concepts, it is generally just another piece of userspace code.

This portion needs some matching "kernel" code which resides inside the UML image
and which implements the Linux kernel part. 

*There are very few limitations on the way "kernel" and "user" interact*. 

UML does not have a strictly defined kernel to host API. It does not try to emulate 
a specific architecture or bus. UML's "kernel" and "user" can share
memory, code and interact as needed to implement whatever design the software developer
has in mind. The only limitations are purely technical. Due to a lot of functions and
variables having the same names, the developer should be careful which includes and libraries
they are trying to refer to.

As a result a lot of userspace code consists of simple wrappers. F.e. `os_close_file()` is
just a wrapper around close() which ensures that the userspace function close does not clash
with similarly named function(s) in the kernel part. 

