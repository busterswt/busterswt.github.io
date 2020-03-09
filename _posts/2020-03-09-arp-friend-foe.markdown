---
title: "IPv4 ARP: Friend or Foe"
layout: post
date: 2020-03-09
image: /assets/images/2020-03-09-arp-friend-foe/friend-or-foe.jpg
headerImage: true
tag:
- openstack
- virtualization
- arp
category: blog
blog: true
author: jamesdenton
description: "Exploring issues with default ARP settings in Linux"
---

I thought I knew ARP. It's known as that protocol that straddles Layer 2 and Layer 3, at the so-called "Layer 2.5" of the OSI model, and helps map IP addresses (Layer 3) to MAC addresses (Layer 2). Simple, right?

Until it isn't.
<!--more-->

ARP is an indispensible component of IPv4 that allows a host to discover hosts on the network and efficiently send traffic to them. Without ARP, you'd be left with broadcasts and a very chatty and slow network.

# The setup

An issue was recently brought to my attention that involved claims of connection timeouts, interruptions, and high latency within the environment. In cases like this, you might start out trying to replicate the issue by sending traffic between multiple instances and looking for dropped packets, latency, etc. to validate the claims. The involvement of virtual network interfaces, such as veths, have been known to introduce these types of issues depending on the kernel version. Insufficient values for "softnet" tunables, such as ```netdev_budget``` and ```netdev_max_backlog``` have also been known to be responsible for dropped packets.

In this case, however, tweaking those values never made a long-term difference.

A look at system resources showed things were running a bit hot, with the `ovs-vswitchd` process consuming 1300-2000% CPU and loads easily over 100 on each node.

A look at the Open vSwitch logs showed a lot of the following:

```
2020-03-06T13:42:05.790Z|165043|timeval(revalidator385)|WARN|context switches: 921 voluntary, 214 involuntary
2020-03-06T13:42:05.915Z|163619|timeval(revalidator383)|WARN|Unreasonably long 4332ms poll interval (3ms user, 1022ms system)
2020-03-06T13:42:05.915Z|163620|timeval(revalidator383)|WARN|faults: 54 minor, 0 major
2020-03-06T13:42:05.915Z|163621|timeval(revalidator383)|WARN|context switches: 995 voluntary, 196 involuntary
2020-03-06T13:42:05.999Z|164849|timeval(revalidator384)|WARN|Unreasonably long 4415ms poll interval (472ms user, 852ms system)
2020-03-06T13:42:05.999Z|164850|timeval(revalidator384)|WARN|faults: 147 minor, 0 major
2020-03-06T13:42:05.999Z|164851|timeval(revalidator384)|WARN|context switches: 986 voluntary, 238 involuntary
2020-03-06T13:42:06.014Z|163920|timeval(revalidator375)|WARN|Unreasonably long 4431ms poll interval (0ms user, 1118ms system)
2020-03-06T13:42:06.014Z|163921|timeval(revalidator375)|WARN|faults: 153 minor, 0 major
2020-03-06T13:42:06.014Z|163922|timeval(revalidator375)|WARN|context switches: 1046 voluntary, 207 involuntary
2020-03-06T13:42:06.035Z|163481|timeval(revalidator382)|WARN|Unreasonably long 4452ms poll interval (560ms user, 826ms system)
2020-03-06T13:42:06.035Z|163482|timeval(revalidator382)|WARN|faults: 121 minor, 0 major
2020-03-06T13:42:06.035Z|163483|timeval(revalidator382)|WARN|context switches: 1008 voluntary, 256 involuntary
2020-03-06T13:42:06.043Z|198653|ofproto_dpif_upcall(revalidator373)|INFO|Spent an unreasonably long 4462ms dumping flows
```

Loss could be seen within OVS on each node:

```
Every 1.0s: ovs-dpctl show | grep lost                                                                                                                          compute5: 

  lookups: hit:7027849315 missed:892873665 lost:28741496
```

And, we were also able to see some dropped packets using `ethtool`:

```
root@compute5:~# ethtool -S enp24s0f1 | egrep -E 'rx_dropped|rx_missed'
     rx_dropped: 3322
     port.rx_dropped: 0
root@compute5:~# ethtool -S enp24s0f0 | egrep -E 'rx_dropped|rx_missed'
     rx_dropped: 768
     port.rx_dropped: 0
```

Packet loss everywhere!

The Intel i40e [tuning guide](https://www.intel.com/content/dam/www/public/us/en/documents/reference-guides/xl710-x710-performance-tuning-linux-guide.pdf) calls out how to address the `rx_dropped` counter, and it's important to note that we were able to stop the increases by adjusting the ring buffer as described in the guide. However, it still didn't address the root issue.

# The trouble with broadcasts

The compute nodes in this environment were loaded to the brim with virtual machine instances all in the same (VLAN) network. In told, there were approximately 716 instances in this network, which itself was a /20. 

Bandwidth utilization did not seem all that high. Using ```bmw-ng```, we were able to see sustained throughput of approximately 2.5Gb/sec on the bond interface. 

A packet capture showed the real story, though:

```
root@compute5:~# for i in {1..5}; do timeout 5s tcpdump -i enp24s0f0 -ne vlan 4034 and arp; sleep 30; done
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp24s0f0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:44:45.625524 fa:16:3e:28:66:0b > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 64: vlan 4034, p 0, ethertype ARP, Request who-has 10.19.50.150 tell 10.19.48.53, length 46
20:44:45.625810 fa:16:3e:7f:9e:d7 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 64: vlan 4034, p 0, ethertype ARP, Request who-has 10.19.48.225 tell 10.19.53.135, length 46
20:44:45.626685 fa:16:3e:6d:2d:c2 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 64: vlan 4034, p 0, ethertype ARP, Request who-has 10.19.50.170 tell 10.19.54.108, length 46
...
20:44:50.509711 fa:16:3e:b0:42:e7 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 64: vlan 4034, p 0, ethertype ARP, Request who-has 10.19.48.124 tell 10.19.53.215, length 46
20:44:50.510249 fa:16:3e:20:fa:3b > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 64: vlan 4034, p 0, ethertype ARP, Request who-has 10.19.49.180 tell 10.19.53.254, length 46
20:44:50.510311 fa:16:3e:bc:48:94 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 64: vlan 4034, p 0, ethertype ARP, Request who-has 10.19.49.55 tell 10.19.52.18, length 46

13595 packets captured
14026 packets received by filter
305 packets dropped by kernel
```

That's a lot of ARP requests! This process repeated 5 times, sleeping 30 seconds after each 5 second capture:

```
16644 packets captured
13826 packets captured
13525 packets captured
14476 packets captured
14320 packets captured
```

With those numbers, we were seeing approximately 2,900 **broadcast** arp requests/second in VLAN 4034 hitting the physical interface of the compute node. Once those ARPs traversed the provider bridge `br-ex` into the integration bridge `br-int`, they would be flooded to each virtual switchport connected to VLAN 4034. Every. Second. 

It's no wonder OVS was running hot!

Using the `-Q` parameter for `tcpdump`, we are able to deduce that the traffic was not being replicated and creating some sort of loop. No, this appeared to be legitimate traffic. But why?

# We must dig deeper

To help narrow down my search for why this was happening, I took a look at one of the broadcast arp requests and the IPs involved:

```
fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
```
In this example, both IPs belonged to two different instances running on two different compute nodes:

```
10.19.49.57 - compute7
10.19.54.136 - compute1
```
A packet capture on the respective tap interface for `10.19.49.57` showed the following:

```
root@compute7:~# tcpdump -i tapa5cd9d43-93 -ne "host 10.19.54.136 and host 10.19.49.57"
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tapa5cd9d43-93, link-type EN10MB (Ethernet), capture size 262144 bytes
18:35:06.109089 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 1006: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 964
18:35:47.644301 fa:16:3e:bd:ef:40 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.19.49.57 tell 10.19.54.136, length 46
18:35:47.644514 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype ARP (0x0806), length 42: Reply 10.19.49.57 is-at fa:16:3e:2e:ee:bd, length 28
18:35:47.645137 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 1025: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 983
18:36:14.894466 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:36:15.897250 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:36:16.904176 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:36:35.075580 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 202: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 160
18:36:35.083793 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1119: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 1077
18:37:03.694710 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:37:03.696223 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype ARP (0x0806), length 60: Reply 10.19.54.136 is-at fa:16:3e:bd:ef:40, length 46
18:37:03.696320 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1010: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 968
18:37:33.894430 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 999: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 957
18:37:37.776202 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 1026: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 984
18:38:31.894371 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:38:31.895631 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1013: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 971
18:38:32.127999 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype ARP (0x0806), length 60: Reply 10.19.54.136 is-at fa:16:3e:bd:ef:40, length 46
18:38:45.696645 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 992: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 950
18:38:58.995058 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1001: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 959
18:39:32.094967 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:39:32.107324 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 987: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 945
18:39:56.706858 fa:16:3e:bd:ef:40 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.19.49.57 tell 10.19.54.136, length 46
18:39:56.706920 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype ARP (0x0806), length 42: Reply 10.19.49.57 is-at fa:16:3e:2e:ee:bd, length 28
18:39:59.841013 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 1027: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 985
18:39:59.841601 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1112: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 1070
18:40:58.172833 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 996: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 954
18:41:08.927350 fa:16:3e:bd:ef:40 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.19.49.57 tell 10.19.54.136, length 46
18:41:08.928095 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype ARP (0x0806), length 42: Reply 10.19.49.57 is-at fa:16:3e:2e:ee:bd, length 28
18:41:09.064741 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 994: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 952
18:41:09.066339 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1079: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 1037
18:41:45.695754 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:41:45.697074 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype ARP (0x0806), length 60: Reply 10.19.54.136 is-at fa:16:3e:bd:ef:40, length 46
18:41:45.707550 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1007: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 965
18:42:57.094524 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:42:57.095415 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype ARP (0x0806), length 60: Reply 10.19.54.136 is-at fa:16:3e:bd:ef:40, length 46
18:42:57.095541 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1032: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 990
18:43:23.294411 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1013: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 971
18:43:41.477281 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype IPv4 (0x0800), length 1011: 10.19.54.136.8301 > 10.19.49.57.8301: UDP, length 969
18:44:20.295244 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:44:21.297736 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:44:22.065782 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 992: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 950
18:44:27.082874 fa:16:3e:bd:ef:40 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.19.49.57 tell 10.19.54.136, length 46
18:44:27.083277 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype ARP (0x0806), length 42: Reply 10.19.49.57 is-at fa:16:3e:2e:ee:bd, length 28
18:44:50.500047 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:44:51.047996 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 984: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 942
```

Traffic out of this port was fairly infrequent, and I began to notice that preceding many of the UDP packets was an ARP request for the destination address:

```
18:41:45.695754 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:41:45.697074 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype ARP (0x0806), length 60: Reply 10.19.54.136 is-at fa:16:3e:bd:ef:40, length 46
18:41:45.707550 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1007: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 965
```

Then, 72 seconds later you see the same thing:

```
18:42:57.094524 fa:16:3e:2e:ee:bd > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.54.136 tell 10.19.49.57, length 28
18:42:57.095415 fa:16:3e:bd:ef:40 > fa:16:3e:2e:ee:bd, ethertype ARP (0x0806), length 60: Reply 10.19.54.136 is-at fa:16:3e:bd:ef:40, length 46
18:42:57.095541 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1032: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 990
```

But, 26 seconds later, you don't:

```
18:43:23.294411 fa:16:3e:2e:ee:bd > fa:16:3e:bd:ef:40, ethertype IPv4 (0x0800), length 1013: 10.19.49.57.8301 > 10.19.54.136.8301: UDP, length 971
```

Without any visibility into the instance itself, it's difficult to determine exactly what was happening. However, some [research](https://serverfault.com/questions/765380/when-do-stale-arp-entries-become-failed-when-never-used) led me to a very important setting within the Linux kernel: `base_reachable_time_ms`.

## base reachable time

In the ARP [manpage](http://man7.org/linux/man-pages/man7/arp.7.html), `base_reachable_time_ms` is described as:

```
Once a neighbor has been found, the entry is considered to be valid for at least a random value between base_reachable_time_ms/2 and 3*base_reachable_time_ms/2.  An entry's validity will be extended if it receives positive feedback from higher level protocols.  Defaults to 30 seconds.
```
This description basically translates to this: An entry in the ARP table is considered REACHABLE for a duration of time between `base_reachable_time_ms/2` and `3*base_reachable_time_ms/2`, at which point it is marked STALE unless there is positive feedback from a higher level protocol (i.e. Layer 4). According to [RFC 1122](https://tools.ietf.org/html/rfc1122), the network stack may attempt to re-validate the entry in various ways before discarding the entry altogether. 

What's important about `base_reachable_time_ms` here is that the default of 30 seconds and range of 15-45 seconds fits perfectly with the provided capture and observed behavior of an ARP request prior to infrequently-sent packets. Without looking at the neighbor table within the instance, it appears that the default value of 30 seconds may have been too aggressive for the application in use.

If this particular instance communicates with other hosts in the same network every 30-60 seconds via UDP, and the ARP cache goes stale before the next packet is sent, then it stands to reason that an ARP request could precede every packet sent to every instance!

Some napkin math shows some possibilities for a single instance:

```
Number of neighbors: 700
Frequency of packets: 30 seconds
Packets per minute: 1,400
Packets per second: 23

Best case (45 second expiration):  3 arps/sec
Worst case (15 second expiration): 11 arps/sec
```
So, in a worst case scenario with each VM instance sending 11 arps/sec, at ~700 instances we can potentially see over 7,000 arps/sec traversing the network at any time! Our best case scenario of 2,100 arps/sec lines up with that we actually saw, though. Keep in mind this is napkin math and subject to assumptions and errors. The packet captures don't lie, though.

# The solution?

Not having a clear picture as to what was going on from an application standpoint, I was able to deduce the following based on the data provided so far:

1. The traffic between an instance and a given neighbor was infrequent enough that the ARP entry became STALE prior to the next packet being sent, which caused the OS to revalidate the entry prior to the next packet.
2. The default value for `base_reachable_time_ms` of 30 seconds is too aggressive for this traffic pattern.

After some back and forth with the application owners, the recommendation was made to adjust `base_reachable_time_ms` from 30000ms (30 seconds) to something like 300000ms (300 seconds) on all virtual machine instances. This change would set the range from 2.5 minutes to 7 minutes, drastically reducing the frequency of ARPs.

Once the changes were made, a later test confirmed these changes moved the needle in the right direction:

```
root@compute4:~# tcpdump -i tape3fce403-bd -ne host 10.19.48.16 and host 10.19.49.12
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tape3fce403-bd, link-type EN10MB (Ethernet), capture size 262144 bytes

14:52:46.686106 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 995: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 953
14:53:03.325773 fa:16:3e:5d:1d:30 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.49.12 tell 10.19.48.16, length 28
14:53:03.329094 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype ARP (0x0806), length 42: Reply 10.19.49.12 is-at fa:16:3e:3a:cf:06, length 28
14:53:03.337863 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1027: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 985
14:53:03.349673 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1107: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 1065
14:53:23.088307 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1007: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 965
14:53:49.087951 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 966: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 924
14:54:04.485889 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1044: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 1002
14:54:14.088263 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1010: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 968
14:54:15.089307 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 979: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 937
14:55:18.688148 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 933: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 891
14:55:20.885037 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 904: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 862
14:56:12.888280 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 876: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 834
14:56:26.088256 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 933: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 891
14:56:31.884918 fa:16:3e:3a:cf:06 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.48.16 tell 10.19.49.12, length 28
14:56:31.885556 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype ARP (0x0806), length 42: Reply 10.19.48.16 is-at fa:16:3e:5d:1d:30, length 28
14:56:31.885648 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 927: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 885
15:01:07.085004 fa:16:3e:3a:cf:06 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.48.16 tell 10.19.49.12, length 28
15:01:07.085982 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype ARP (0x0806), length 42: Reply 10.19.48.16 is-at fa:16:3e:5d:1d:30, length 28
15:01:07.086361 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 995: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 953
15:01:12.485582 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 963: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 921
15:01:13.288546 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 988: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 946
15:01:13.288598 fa:16:3e:5d:1d:30 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.19.49.12 tell 10.19.48.16, length 28
15:01:13.296555 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype ARP (0x0806), length 42: Reply 10.19.49.12 is-at fa:16:3e:3a:cf:06, length 28
15:01:15.285880 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1014: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 972
15:01:16.613040 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 981: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 939
15:01:17.115247 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1015: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 973
15:01:18.485507 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1032: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 990
15:01:37.290410 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1015: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 973
15:01:54.484909 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 991: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 949
15:01:55.365367 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1034: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 992
15:01:55.369380 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1098: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 1056
15:02:11.643067 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 999: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 957
15:02:11.663588 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 998: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 956
15:02:12.164886 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 981: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 939
15:02:16.885188 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1052: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 1010
15:02:16.888281 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1097: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 1055
15:02:25.178396 fa:16:3e:5d:1d:30 > fa:16:3e:3a:cf:06, ethertype IPv4 (0x0800), length 1042: 10.19.48.16.8301 > 10.19.49.12.8301: UDP, length 1000
15:02:46.484979 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 932: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 890
15:02:48.286197 fa:16:3e:3a:cf:06 > fa:16:3e:5d:1d:30, ethertype IPv4 (0x0800), length 1004: 10.19.49.12.8301 > 10.19.48.16.8301: UDP, length 962
```

ARPs could still be seen, but on a much less-frequent basis. The `base_reachable_time_ms` could be further adjusted, depending on the application, especially if instances were deployed together and there's no expectation that IP<->MAC relationships would change throughout the life of the application. This reduction in 'arp noise' greatly reduced the cycles spent on Open vSwitch, which in turn helped introduce stability into the environment. 

The lack of unicast ARP polling, as described in [RFC 1122](https://tools.ietf.org/html/rfc1122#section-2.3.2.1) Section 2.3.2.1, really hurt us here as well, since every ARP request was blasted as a broadcast. Further research is also needed to determine whether UDP traffic serves as "higher-layer advice" for validating ARP entries, or if that is left to TCP only.

If you have some thoughts or comments on this process, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.