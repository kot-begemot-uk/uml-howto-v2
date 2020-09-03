# Running UML

This section assumes that either the
user-mode-linux package from the distribution or a custom built kernel has been installed
on the host. 

These add an executable called linux to the system. This
is the UML kernel. It can be run just like any other executable.
It will take most normal linux kernel arguments as command line
arguments.

Additionally, it will need some UML specific arguments 
in order to do something useful.

### Mandatory Arguments:

1. mem= memory[K,M,G] - amount of memory. By default bytes. It will
also accept K, M or G qualifiers. 
1. ubdX[s,d,c,t]= virtual disk specification. This is not really
mandatory, but it is likely to be needed in nearly all cases so we can specify 
a root file system.
    1. The simplest possible image specification is the name of the image file for the
    filesystem (created using one of the methods described in Generating an Image).
    1. UBD devices support copy on write (COW). The changes are kept in a separate
    file which can be discarded allowing a rollback to the original pristine image.
    If COW is desired, the UBD image is specified as: cow_file,master_image. Example:`ubd0=Filesystem.cow,Filesystem.img`
    1. UBD devices can be set to use synchronous IO. Any writes are immediately flushed
    to disk. This is done by adding `s` after the ubdX specification
    1. UBD performs some euristics on devices specified as a single filename to make sure
    that a COW file has not been specified as the image. To turn them off, use the d flag after ubdX
    1. UBD supports TRIM - asking the Host OS to reclaim any unused blocks in the image. To turn it off,
    specify the t flag after ubdX
1. root= root device - most likely /dev/ubd0 (this is a Linux filesystem image)

### Important Optional Arguments

If UML is run as "linux" with no extra arguments, it will try to start an xterm for every console
configured inside the image (up to 6 in most linux distributions). Each console is started inside an
xterm. This makes it nice and easy to use UML on a host with a GUI. It is, however,
the wrong approach if UML is to be used as a testing harness or run in 
a text-only environment.

In order to change this behaviour we need to specify an alternative console
and wire it to one of the supported "line" channels. For this we need to map a
console to use something different from the default
xterm.

Example which will divert console number 1 to stdin/stdout:
```
con1=fd:0,fd:1
```

UML supports a wide variety of serial line channels which are specified using the
following syntax
``
conX=channel_type:options[,channel_type:options]
``
If the channel specification contains two parts separated by comma, the first one
is input, the second one output.

1. The null channel - Discard all input or output. Example `con=null` will set all consoles to null by default.
1. The fd channel - use file descriptor numbers for input/out. Example: `con1=fd:0,fd:1.`
1. The port channel - listen on tcp port number. Example: `con1=port:4321`
1. The pty and pts channels - use system pty/pts. 
1. The tty channel - bind to an existing system tty. Example: `con1=/dev/tty8` will make UML use the host 8th console (usually unused).
1. The xterm channel - this is the default - bring up an xterm on this channel and direct IO to it. Note, that in order for xterm to work,
the host must have the UML distribution package installed. This usually contains the port-helper and other utilities needed for UML to
communicate with the xterm. Alternatively, these need to be complied and installed from source.

All options applicable to consoles also apply to UML serial lines which are presented as ttyS inside UML.

## Starting UML

We can now run UML.

```
linux mem=2048M umid=TEST \
    ubd0=Filesystem.img \
    vec0:transport=tap,ifname=tap0,depth=128,gro=1 \
    root=/dev/ubda con=null con0=null,fd:2 con1=fd:0,fd:1
```

This will run an instance with 2048M RAM, try to use the image file called Filesystem.img as root. 
It will connect to the host using tap0. All consoles except con1 will be disabled and console 1 will
use standard input/output making it appear in the same terminal it was started.

## Logging in

If you have not set up a password when generating the image, you will have to shut down the UML
instance, mount the image, chroot into it and set it - as described in the Generating an Image section.

If the password is already set, you can just log in.

## The UML Management Console

In addition to managing the image from "the inside" using normal sysadmin tools, it is possible
to perform a number of low level operations using the UML management console.

The UML management console is a low-level interface to the kernel on a running UML instance,
somewhat like the i386 SysRq interface.  Since there is a full-blown
operating system under UML, there is much greater flexibility possible
than with the SysRq mechanism.


There are a number of things you can do with the mconsole interface:

* get the kernel version
* add and remove devices
* halt or reboot the machine
* Send SysRq commands
* Pause and resume the UML
* Inspect processes running inside UML
* Inspect UML internal /proc state


You need the mconsole client (uml\_mconsole) which is a part of the UML
tools package available in most Linux distritions.
You also need `CONFIG_MCONSOLE` (under 'General Setup') enabled in the UML kernel.  When you boot UML, you'll see a line like:

```
mconsole initialized on /home/jdike/.uml/umlNJ32yL/mconsole
```

If you specify a unique machine id one the UML command line, i.e.

```
umid=debian
```

you'll see this

```
mconsole initialized on /home/jdike/.uml/debian/mconsole
```

That file is the socket that uml_mconsole will use to communicate with
UML.  Run it with either the umid or the full path as its argument:

```
host$ uml_mconsole debian
```

or

```
host$ uml_mconsole /home/jdike/.uml/debian/mconsole
```

You'll get a prompt, at which you can run one of these commands:

* version
* help
* halt
* reboot
* config
* remove
* sysrq
* help
* cad
* stop
* go
* proc
* stack

### version

This takes no arguments.  It prints the UML version.

```
(mconsole)  version
OK Linux OpenWrt 4.14.106 #0 Tue Mar 19 08:19:41 2019 x86_64
```

There are a couple actual uses for this.  It's a simple no-op which
can be used to check that a UML is running.  It's also a way of
sending a device interrupt to the UML. UML mconsole is treated internally as
a UML device. 

### help

This takes no arguments. It prints a short help screen with the
supported mconsole commands.


### halt and reboot

These take no arguments.  They shut the machine down immediately, with
no syncing of disks and no clean shutdown of userspace.  So, they are
pretty close to crashing the machine.

```
(mconsole)  halt
OK
```


# config

"config" adds a new device to the virtual machine. This is supported
by most UML device drivers. It takes one argument, which is the
device to add, with the same syntax as the kernel command line.

```
(mconsole)
config ubd3=/home/jdike/incoming/roots/root_fs_debian22
OK
```

### remove

"remove" deletes a device from the system.  Its argument is just the
name of the device to be removed. The device must be idle in whatever
sense the driver considers necessary.  In the case of the ubd driver,
the removed block device must not be mounted, swapped on, or otherwise
open, and in the case of the network driver, the device must be down.

```
(mconsole)  remove ubd3
OK
(mconsole)  remove eth1
OK
```

### sysrq

This takes one argument, which is a single letter.  It calls the
generic kernel's SysRq driver, which does whatever is called for by
that argument.  See the SysRq documentation in
Documentation/admin-guide/sysrq.rst in your favorite kernel tree to
see what letters are valid and what they do.



### cad

This invokes the Ctl-Alt-Del action in the running image.  What exactly this ends
up doing is up to init, systemd, etc.  Normally, it reboots the machine.

### stop

This puts the UML in a loop reading mconsole requests until a 'go'
mconsole command is received. This is very useful as a debugging/snapshotting
tool.

### go

This resumes a UML after being paused by a 'stop' command. Note that
when the UML has resumed, TCP connections may have timed out and if
the UML is paused for a long period of time, crond might go a little
crazy, running all the jobs it didn't do earlier.

### proc

This takes one argument - the name of a file in /proc which is printed
to the mconsole standard output

### stack

This takes one argument - the pid number of a process. Its stack is
printed to a standard output.


