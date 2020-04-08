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
1. root= root device - most likely /dev/ubd0 (this is a Linux 

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
1. The xterm channel - this is the default on - bring up an xterm on this channel and direct IO to it. Note, in order for xterm to work,
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
