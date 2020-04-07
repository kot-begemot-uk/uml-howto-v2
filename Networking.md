# UML Networking

UML networking is designed to emulate an Ethernet connection. This
connection may be either a point-to-point (similar to a connection
between machines using a back-to-back cable) or a connection to a
switch. UML supports a wide variety of means to build these
connections to all of: local machine, remote machine(s), local and 
remote UML and other VM instances.

Supported transports (as of kernel 5.6)

Transport | Type |Capabilities | Speed (on 3.5GHz Ryzen)
----------|------|-------------|---------------------------------
tap | vector | checksum and tso offloads | > 8GBit
hybrid | vector | checksum and tso offloads, multipacket rx | > 6GBit
raw | vector | checksum and tso, offloads, multipacket rx, tx | > 6GBit
Ethernet over gre | vector | multipacket rx, tx | > 3Gbit
Ethernet over l2tpv3 | vector | multipacket rx, tx | > 3Gbit
bess | vector | multipacket rx, tx | > 3Gbit
tuntap | legacy | none | ~ 500Mbit
daemon | legacy | none | ~ 450Mbit
socket | legacy | none | ~ 450Mbit
pcap | legacy | none | ~400Mbit, rx only
ethertap | obsolete | none | uknown
vde | obsolete | none | uknown

* All transports which have tso and checksum offloads can deliver speeds
approaching 10G on TCP streams. 
* All transports which have multi-packet rx and/or tx can deliver pps rates of
up to 1Mps or more.
* All legacy transports are generally limited to ~600-700MBit and 0.05Mps
* GRE and L2TPv3 allow connections to all of: local machine, remote machines, remote network devices and remote UML instances.
* Socket allows connections only between UML instances.
* Daemon and bess require running a local switch. This switch may be connected to the host as well.


## Configuring vector transports

All vector transports support a similar syntax:

If X is the interface number as in vec0, vec1, vec2, etc, the general syntax
for options is:

vecX:transport="Transport Name",option=value,option=value,...,option=value

These options are common for all transports:
1. depth = int - sets the queue depth for vector IO. This is the amount of packets UML will attempt to read or write in a single system call. The default number
is 64 and is generally sufficient for most ~ 2-4G applications. Higher speeds
may require larger values.
1. mac ="XX:XX:XX:XX:XX" - sets the interface MAC value.
1. gro = 0/1 - sets GRO on or off. The offload which is most commonly enabled
as a result of this option being on is TCP if it is possible. Note, that it
is usually enabled by default on local machine interfaces (f.e. veth pairs),
so it should be enabled for a lot of transports for networking to operate
correctly.
1. mtu = int - sets the interface mtu
1. headroom = int - adjusts the default headroom (32 bytes) reserved if a packet
will need to be re-encapsulated into let's say VXLAN.


Transports which bind to a local network interface share another option - the
name of the interface to bind to. This is the ifname option. 

### tap transport

Example:
```shell
vecX:transport=tap,ifname=tap0,depth=128,gro=1
```
This will connect vec0 to tap0 on the host. Tap0 must be created (f.e. using
tunctl) and UP.

tap0 can be either configured as a point-to-point interface and given ip
address so that UML can talk to the host. Alternatively, it is possible to
connect UML to a tap interface which is a part of the bridge.

While tap relies on the vector infrastructure it is not a true vector transport
at this point because Linux does not support multi-packet IO on tap
file descriptors for normal userspace apps like UML. This is a privilege
which is offered only to something which can hook up to it at kernel level
via specialized interfaces like vhost-net. A similar helper for UML is planned
at some point in the future.

### hybrid transport

Example:
```shell
vecX:transport=hybrid,ifname=tap0,depth=128,gro=1
```
This is an experimental/demo transport which couples tap for transmit and a raw socket for receive. The raw socket allows multi-packet receive resulting in
significantly higher packet rates than normal tap

### raw socket transport
Example:
```shell
vecX:transport=raw,ifname=veth0,depth=128,gro=1
```
This transport uses vector IO on raw sockets. While you can bind to any
interface including a physical one, the most common use it to bind to
the "peer" side of a veth pair with the other side configured on the host.

Example host configuration for Debian:
```
#/etc/network/interfaces
auto veth0
iface veth0 inet static
        address 192.168.4.1
        netmask 255.255.255.252
        broadcast 192.168.4.3
        pre-up ip link add veth0 type veth peer name p-veth0 && ifconfig p-veth0 up
```
UML can now bind to p-veth0 like this:
```shell
vec0:transport=raw,ifname=p-veth0,depth=128,gro=1
```
If the UML guest is configured with 192.168.4.2 and netmask 255.255.255.0
it can talk to the host on 192.168.4.1

The raw transport also provides some support for offloading some of the
filtering to the host. The two options to control it are:

1. bpffile = filename of raw bpf code to be loaded as a socket filter
1. bpfflash = 0/1 allow loading of bpf from inside User Mode Linux. This option
allows the use of the ethtool load firmware command to load bpf code. 

In either case the bpf code is loaded into the host kernel. While this is
presently limited to legacy bpf syntax (not ebpf), it is still a security
risk. It is not recommended to allow this unless the User Mode Linux instance
is considered trusted.

### gre socket transport
Example:
```shell
vecX:transport=gre,src=src_host,dst=dst_host 
```

This will configure an Ethernet over GRE (aka GRETAP or GREIRB) tunnel which
will connect the UML instance to a GRE endpoint at host dst\_host.

GRE supports the following additional options:

1. rx\_key= - gre 32 bit integer key for rx packets, if set, tx\key must be set too
1. tx\_key= - gre 32 bit integer key for tx packets, if set rx\_key must be set too
1. sequence = 0/1- enable gre sequence
1. pin\_sequence = 0/1- pretend that the sequence is always reset on each packet
(needed to interop with some really broken implementations)
1. v6 = 0/1 - force v6 sockets
1. Gre checksum is not presently supported

GRE has a number of caveats:
1. You can use only one gre connection per ip address. There is no way to
multiplex connections as each GRE tunnel is terminated directly on the UML
instance.
1. The key is not really a security feature. While it was intended as such
it's "security" is laughable. It is, however, a useful feature to ensure
that the tunnel is not misconfigured.

An example configuration for a Linux host with a local address of 192.168.128.1
to connect to a UML instance at 192.168.129.1

```
#/etc/network/interfaces
auto gt0
iface gt0 inet static
address 10.0.0.1
netmask 255.255.255.0
broadcast 10.0.0.255
mtu 1500
pre-up ip link add gt0 type gretap local 192.168.128.1 remote 192.168.129.1 || true
down ip link del gt0 || true
```

Additionally, GRE has been tested versus a variety of network equipment.

### l2tpv3 socket transport
_Warning_. L2TPv3 has a "bug". It is the "bug" known as "has more options than GNU ls". While it has some advantages, there are usually easier (and less verbose) ways to connect a UML instance to something. Fir example, most devices which support L2TPv3
also support GRE.

Example:
```shell
vec0:transport=l2tpv3,udp=1,src=$src_host,dst=$dst_host,srcport=$src_port,dstport=$dst_port,depth=128,rx_session=0xffffffff,tx_session=0xffff
```

This will configure an Ethernet over L2TPv3 fixed tunnel which
will connect the UML instance to a L2TPv3 endpoint at host $dst\_host using
the L2TPv3 UDP flavour and UDP destination port $dst\_port.

GRE always requires the following additional options:

1. rx\_session= - l2tpv3 32 bit integer session for rx packets -
1. tx\_session= - l2tpv3 32 bit integer session for tx packets

As the tunnel is fixed these are not negotiated and they are preconfigured
on both ends.

Additionally, L2TPv3 supports the following optional parameters


1. rx\_cookie= - l2tpv3 32 bit integer cookie for rx packets - same
functionality as GRE key, more to prevent misconfiguration than provide
actual security
1. tx\_cookie= - l2tpv3 32 bit integer cookie for tx packets
1. cookie64 = 0/1 - use 64 bit cookies instead of 32 bit.
1. counter = - enable l2tpv3 counter 
1. pin\_counter - pretend that the counter is always reset on each packet
(needed to interop with some really broken implementations)
1. v6 = 0/1 - force v6 sockets
1. udp = 0/1 - use raw sockets (0) or UDP (1) version of the protocol

L2TPv3 has a number of caveats:
1. you can use only one connection per ip address in raw mode. There is
no way to multiplex connections as each L2TPv3 tunnel is terminated
directly on the UML instance. UDP mode can use different ports for this purpose.

Here is an example of how to configure a linux host to connect to UML via L2TPv3:

```shell
auto l2tp1
iface l2tp1 inet static
address 192.168.126.1
netmask 255.255.255.0
broadcast 192.168.126.255
mtu 1500
pre-up ip l2tp add tunnel remote 127.0.0.1 local 127.0.0.1 encap udp tunnel_id 2 peer_tunnel_id 2 udp_sport 1706 udp_dport 1707 && ip l2tp add session name l2tp1 tunnel_id 2 session_id 0xffffffff peer_session_id 0xffffffff
down ip l2tp del session tunnel_id 2 session_id 0xffffffff && ip l2tp del tunnel tunnel_id 2
```

### BESS socket transport

BESS is a high performance modular network switch. 

https://github.com/NetSys/bess

It has support for a simple
sequential packet socket mode which in the more recent versions is using
vector IO for high performance.

Example:
```shell
vecX:transport=bess,src=unix_src,dst=unix_dst
```

This will configure a BESS transport using the unix\_src Unix domain socket
address as source and unix\_dst socket address as destination.

For BESS configuration and how to allocate a BESS Unix domain socket port
please see the BESS documentation.

https://github.com/NetSys/bess/wiki/Built-In-Modules-and-Ports


