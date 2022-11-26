---
title: "LXD, One Bridge, Multiple VLANs"
date: 2022-11-12T01:01:01-00:01
categories:
  - blog
tags:
  - LXD
  - Networking
---
I'm old and scared of NetworkManager, so when installing the primary server
on my home network the newer methods of network configuration were axed, and
/etc/network/interfaces reinstated to its former glory.  Since every bit of
kit was on the same subnet, it made sense to just create a bridge on a fixed
ip, then start up the lxd containers on the bridge and let them go on their
merry way.

Of course now that I have the brocade, I need to split the containers by
vlan to be available for all users (plex), non-guest users (samba) and just
me (linux iso downloading apps).  Initial googling suggested a whole bunch
of new vlan interfaces, and bridges on each, which seemed dumb.  Deeper
googling suggested that vlan-aware bridges are a thing, and
VLANFiltering=true is probably the magic incantation.

To be clear, I want one bridge with access to several bridged vlans. I do
not want to create a bunch of individual vlans and bridges.

Further sleuthing reveals a crossroads-  one can either migrate from ifupdown
to ifupdown2, which has drop-in bridging for this use case, or go back to
native ubuntu tools (NetworkManager [gah!] or systemd-networkd.  I choose
the latter for no reason but it would be fun to rename the interfaces by
physical location [top, bottom, etc, but 'test0' for this exercise]).



This [helpful
discussion](https://discuss.linuxcontainers.org/t/lxd-containers-on-a-vlan-aware-bridge/14734/2)
 isn't quite enough to infer the correct incantations without
bonding multiple interfaces, but the man pages for [systemd.link](https://www.freedesktop.org/software/systemd/man/systemd.link.html),
[systemd.netdev](https://www.freedesktop.org/software/systemd/man/systemd.netdev.html),
and
[systemd.network](https://www.freedesktop.org/software/systemd/man/systemd.netdev.html)
can get us sorted.  Briefly:

1.  Our 10-testlan.link file looks like this:
```yaml
[Match]
#OriginalName=enp3s0
MACAddress=a8:a1:59:3f:b2:7b
[Link]
Description="Middle Port"
Name=test0
```

2. 15-test0.network:
```yaml
[Match]
Name=test0

[Network]
Bridge=tbr0

[BridgeVLAN]
VLAN=2-4094
```

3. Our 20-testbridge.netdev file looks like this:
```yaml
[NetDev]
Name=tbr0
Kind=bridge

[Bridge]
DefaultPVID=1
VLANFiltering=true
STP=false
```

4. ...and 30-testbridge.network
```yaml
[Match]
Name=tbr0

[Network]
VLAN=direct

[BridgeVLAN]
VLAN=2-4094
```

Ideally, after rebooting, we will still have the working bridge (br0, with static
address) and a new bridge (no address, but vlan aware).  Simply starting the
service wasn't enough, and debugging the wait-online service seems like a
pain.  It seems to work:

```
root@perineum:~# ip addr show
<snip>
3: test0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master tbr0 state UP group default qlen 1000
    link/ether a8:a1:59:3f:b2:7b brd ff:ff:ff:ff:ff:ff
4: tbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether a8:a1:59:3f:b2:7b brd ff:ff:ff:ff:ff:ff
    inet6 fe80::aaa1:59ff:fe3f:b27b/64 scope link
       valid_lft forever preferred_lft forever
```
Now lets launch a new lxd container on the new bridge and see if it picks up an
ip address on the right subnet. VLAN 100 is already configured here for dhcp, so
we just add the port on the brocade switch to the right vlan:

```
SSH@chonk(config)#vlan 100
SSH@chonk(config-vlan-100)#tagged eth 1/1/16
Added tagged port(s) ethe 1/1/16 to port-vlan 100.
```

With hints from [Simon
Gillet](https://somm15.github.io/network/vlan/lxd/2018/12/13/vlan_with_lxd.html)
and [Platu](https://gist.github.com/platu/fc0c22a42d002a00382c20e023658688) we
create a new lxd profile for vlan 100. Ruhroh:
```
root@perineum:~# lxc profile copy default vlan100
root@perineum:~# lxc profile device set vlan100 eth0 vlan 100
Error: Device validation failed for "eth0": Invalid device option "vlan"
```
The [first google
result](https://discuss.linuxcontainers.org/t/ovs-vlan-tag-with-lxd/13984/3)
suggests a version check.
```
root@perineum:~# lxd --version
4.0.9
```
So lets upgrade lxd from 4.0.9 to 5.0 (LTS) and try again.
```
root@perineum:~# snap refresh lxd --channel=5.0/stable
2022-11-19T22:43:45-05:00 INFO Waiting for "snap.lxd.daemon.service" to stop.
lxd (5.0/stable) 5.0.1-9dcf35b from Canonical✓ refreshed
root@perineum:~# lxc profile device set vlan100 eth0 vlan 100
root@perineum:~# lxc profile device set vlan100 eth0 parent tbr0
root@perineum:~# lxc launch ubuntu:20.04 bridgetest --profile vlan100
root@perineum:~# lxc list
+------------+---------+------------------------+------+-----------+-----------+
|    NAME    |  STATE  |          IPV4          | IPV6 |   TYPE    | SNAPSHOTS |
+------------+---------+------------------------+------+-----------+-----------+
| bridgetest | RUNNING | 192.168.100.102 (eth0) |      | CONTAINER | 0         |
+------------+---------+------------------------+------+-----------+-----------+
```
Success!
