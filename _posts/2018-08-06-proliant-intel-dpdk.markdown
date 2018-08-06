---
title: "DPDK and RMRR Compatibility Issues on the HP Proliant DL360e G8"
layout: post
date: 2018-08-06
image: /assets/images/2018-08-06-proliant-intel-dpdk/detour.png
headerImage: true
tag:
- blog
- neutron
- ovs
- dpdk
- openstack
- proliant
- dl360e
- ubuntu
- openstack-ansible
blog: true
author: jamesdenton
description: DPDK and RMRR Compatibility Issues on the HP Proliant DL360e G8
---

Earlier this year I made it a goal to spend more time on network virtualization technologies and software-defined networking, and have recently found myself attempting to build out an OpenStack environment using the Open vSwitch ML2 mechanism driver with support for DPDK.

<!--more-->
For those that don't know, **DPDK** stands for **Data Plane Development Kit**, and is a set of libraries and network controller drivers for fast packet processing. These drivers, and applications that support them (such as OVS), allow network traffic to bypass the kernel in an attempt to decrease latency and increase the number of packets that can be processed by the host. Originally envisioned by Intel, DPDK is now an open-source project under the Linux Foundation umbrella. More information can be found [here](https://www.dpdk.org). 

I can't do it justice here, but there are many good resources on DPDK and its advantages. Below are some resources worth checking out:

- [https://blog.selectel.com/introduction-dpdk-architecture-principles/](https://blog.selectel.com/introduction-dpdk-architecture-principles/)
- [https://media.readthedocs.org/pdf/dpdk/latest/dpdk.pdf](https://media.readthedocs.org/pdf/dpdk/latest/dpdk.pdf)

My goal at the start of this effort was simple: Figure out what was needed to supplement the existing Open vSwitch deployment mechanism in OpenStack-Ansible to support DPDK. I opened up a [bug](https://bugs.launchpad.net/openstack-ansible/+bug/1784660) and away I went.

# Getting started

The following hardware was used:

```
HP Proliant DL360e G8 - 4x LFF Slots
1x 2TB Hitachi 7200rpm SATA Drive
96GB RAM
Intel X520 2-port 10-Gigabit Ethernet Network Card
Ubuntu 16.04 LTS Operating System
```

The NIC in question is an Intel X520 82599ES-based 2x10G Network Interface Card that operates in a PCI 2.0 or 3.0 slot. This one in particular has 2x SFP+ interfaces using non-Intel SFP+ modules, mainly because they're all I have. The card is non-OEM, which means there are no readily-available ROM updates available, especially from HP. ROM updates aren't needed here, but it's worth pointing out since many folks often turn to firmware and drivers when things don't work as we think they should.


