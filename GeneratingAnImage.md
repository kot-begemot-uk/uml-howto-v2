# Building a UML instance

There is no UML installer in any distribution. While you can use off
the shelf install media to install into a blank VM using a virtualization
package, there is no UML equivalent. You have to use appropriate tools on
your host to build a viable filesystem image.

This is extremely easy on Debian - you can do it using debootstrap. It is
also easy on OpenWRT - the build process can build UML images. All other
distros - YMMV.

# Creating an image using debootstrap.

1.Create a sparse raw disk image:
`dd if=/dev/zero of=disk_image_name bs=1 count=1 seek=16G` 
will create a 16G disk image. The OS will initially allocate only one block and will allocate more as they are written by UML. As of kernel  version 4.19 UML fully supports TRIM (as usually used by flash drives). Using
TRIM inside the UML image by specifying discard as a mount option or by
running tune2fs -o discard /dev/ubdXX will request UML to return any unused
blocks to the OS. 

## Create a filesystem on the disk image and mount it:
```shell
mkfs.ext4 ./disk_image_name
mount ./disk_image_name /mnt
```
This example uses ext4, any other filesystem such as ext3, btrfs, xfs,
jfs, etc will work too.

## Create a minimal OS installation on the mounted filesystem
  
```shell
debootstrap buster /mnt http://deb.debian.org/debian
```
debootstrap does not set up the root password, fstab, hostname or anything
related to networking. It is up to the user to do that.

## Set the root password
The easiest way to do that is to chroot to the mounted image.
```shell
  chroot /mnt
  passwd
  exit
```
## Edit key system files

### Edit fstab

UML block devices are called ubds. The fstab created by debootstrap will be
empty and it needs an entry for the root file system:

```
  /dev/ubd0               ext4    discard,errors=remount-ro  0       1
```
  
### Edit hostname. 

It will be set to the same as the host on which you 
  are creating the image. It is a good idea to change that to avoid "Oh,
  bummer, I rebooted the wrong machine".
 
### Edit network/interfaces on Debian or sysconfig/network-scripts/ifcfg-X on Red Hat

UML supports two classes of network devices - the older uml\_net ones which
are scheduled for obsoletion. These are called ethX. It also suports the 
newer vector IO devices which are significantly faster and have support
for some standard network encapsulations like Ethernet over GRE and 
Ethernet over L2TPv3. These are called vec0.

Depending on which one is in use, /etc/network/interfaces will need an entry
like 
```
    auto eth0
    iface eth0 inet dhcp
```
  or
```
    auto vec0
    iface eth0 inet dhcp
``` 

We now have a UML image which is nearly ready to run, all we need is a UML
kernel and modules for it.

Most distributions have a UML package. Even if you intend to use your own
kernel, testing the image with a stock one is always a good start. These
packages come with a set of modules which should be copied to the target
filesystem. The location is distribution dependent. For Debian these reside
under /usr/lib/uml/modules. Copy recursively the content of this directory
to the mounted UML filesystem:
```shell
cp -rax /usr/lib/uml/modules /mnt/lib/modules
```
If you have compiled your own kernel, you need to
use the usual "install modules to a location" procedure by running

```shell
  make install MODULES_DIR=/mnt/lib/modules
```

At this point the image is ready to be brought up.
