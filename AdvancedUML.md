# Advanced UML Topics

## Sharing Filesystems between Virtual Machines

Don't attempt to share filesystems simply by booting two UMLs from the
same file.  That's the same thing as booting two physical machines
from a shared disk.  It will result in filesystem corruption.

### Using layered block devices

The way to share a filesystem between two virtual machines is to use
the copy-on-write (COW) layering capability of the ubd block driver.
Any changed blocks are stored
in the private COW file, while reads come from either device - the
private one if the requested block is valid in it, the shared one if
not.  Using this scheme, the majority of data which is unchanged is
shared between an arbitrary number of virtual machines, each of which
has a much smaller file containing the changes that it has made.  With
a large number of UMLs booting from a large root filesystem, this
leads to a huge disk space saving. 

It will also help performance,
since the host will be able to cache the shared data using a much
smaller amount of memory, so UML disk requests will be served from the
host's memory rather than its disks.

There is a major caveat in doing this on multisocket NUMA machines. On such
hardware, running many UML instances with a shared master image and
COW changes may kill the machine and/or start putting CPUs offline due to
NMIs from excess of inter-socket traffic. If you are running UML on high
end hardware like this, make sure to bind it to cpus residing on the same
socket using the `taskset` command or have a look at the "tuning" section.

To add a copy-on-write layer to an existing block device file, simply
add the name of the COW file to the appropriate ubd switch:

```
ubd0=root_fs_cow,root_fs_debian_22
```

where 'root_fs_cow' is the private COW file and 'root_fs_debian_22' is
the existing shared filesystem.  The COW file need not exist.  If it
doesn't, the driver will create and initialize it.

### Disk Usage

UML has TRIM support which will release any unused space in its disk image files to
the underlying OS. It is important to use either ls -ls or du to verify the actual
file size.

### COW validity.

Any changes to the master image will invalidate all COW files. If this happens, UML will *NOT* automatically
delete any of the COW files and will refuse to boot. In this case the only solution is to either restore the old image (including its last modified timestamp) or remove all COW files which will result in their recreation. Any changes in the COW files will be lost.

### Cows can moo - uml_moo : Merging a COW file with its backing file

Depending on how you use UML and COW devices, it may be advisable to
merge the changes in the COW file into the backing file every once in
a while.

The utility that does this is uml_moo.  Its usage is

```
uml_moo COW_file new_backing_file
```

There's no need to specify the backing file since that information is
already in the COW file header.  If you're paranoid, boot the new
merged file, and if you're happy with it, move it over the old backing
file.

uml_moo creates a new backing file by default as a safety measure.  It
also has a destructive merge option which will merge the COW file
directly into its current backing file.  This is really only usable
when the backing file only has one COW file associated with it.  If
there are multiple COWs associated with a backing file, a -d merge of
one of them will invalidate all of the others.  However, it is
convenient if you're short of disk space, and it should also be
noticeably faster than a non-destructive merge.

uml_moo is installed with the UML distribution packages and is available as a part
of UML utilities.

## Host file access

If you want to access files on the host machine from inside UML, you
can treat it as a separate machine and either nfs mount directories
from the host or copy files into the virtual machine with scp or rcp.

However, since UML is running on the host, it can access those
files just like any other process and make them available inside the
virtual machine without needing to use the network.

This is possible with the hostfs virtual filesystem.  With it, you
can mount a host directory into the UML filesystem and access the
files contained in it just as you would on the host.


*SECURITY WARNING*

Hostfs without any parameters to the UML Image will allow the image to mount any
part of the host filesystem and write to it. Always confine hostfs to a specific
"harmless" directory (f.e. `/var/tmp`) if running UML. This is especially important
if UML is being run as root.

### Using hostfs

To begin with, make sure that hostfs is available inside the virtual
machine with

```
UML# cat /proc/filesystems
```

`hostfs` should be listed.  If it's not, either rebuild the kernel
with hostfs configured into it or make sure that hostfs is built as a
module and available inside the virtual machine, and insmod it.


Now all you need to do is run mount:

```
UML# mount none /mnt/host -t hostfs
```



will mount the host's / on the virtual machine's /mnt/host.


If you don't want to mount the host root directory, then you can
specify a subdirectory to mount with the -o switch to mount:

```
UML# mount none /mnt/home -t hostfs -o /home
```

will mount the hosts's /home on the virtual machine's /mnt/home.


### hostfs as the root filesystem

It's possible to boot from a directory hierarchy on the host using
hostfs rather than using the standard filesystem in a file.

To start, you need that hierarchy.  The easiest way is to loop mount
an existing root_fs file:

```
host#  mount root_fs uml_root_dir -o loop
```

You need to change the filesystem type of / in etc/fstab to be
'hostfs', so that line looks like this:

```
/dev/ubd/0       /        hostfs      defaults          1   1
```

Then you need to chown to yourself all the files in that directory
that are owned by root.  This worked for me:

```
host#  find . -uid 0 -exec chown jdike {} \;
```

Next, make sure that your UML kernel has hostfs compiled in, not as a
module.  Then run UML with the boot device pointing at that directory:

```
ubd0=/path/to/uml/root/directory
```

UML should then boot as it does normally.

### Hostfs Caveats

Hostfs does not support keeping track of host filesystem changes on the host (outside UML). As a result,
if a file is changed without UML's knowledge, UML will not know about it and its own in-memory cache of
the file may be corrupt. While it is possible to fix this, it is not something which is being worked on
at present.

## Tuning UML

UML at present is strictly uniprocessor. It will, however spin up a number of threads to handle various functions.

The UBD driver, SIGIO and the MMU emulation spin up threads. If the system is idle, these threads will be
migrated to other processors on a SMP host. This, unfortunately, will usually result in LOWER performance because of
all of the cache/memory synchronization traffic between cores. As a result, UML will usually benefit from being pinned on
a single CPU especially on a large system. This can result in performance differences of 5 times or higher on
some benchmarks. 

Similarly, on large multi-node NUMA systems UML will benefit if all of its memory is allocated from the same NUMA node it will
run on. The OS will *NOT* do that by default. In order to do that, the sysadmin needs to create a suitable tmpfs
ramdisk bound to a particular node and use that as the source for UML RAM allocation by specifying it  in the TMP or TEMP
environment variables. UML will look at the values of TMPDIR, TMP or TEMP 
for that. If that fails, it will look for shmfs mounted under /dev/shm. If everything else fails use /tmp/ regardless of the filesystem
type used for it.

```
mount -t tmpfs -ompol=bind:X none /mnt/tmpfs-nodeX
TEMP=/mnt/tmpfs-nodeX taskset -cX linux options options options.. 
```

