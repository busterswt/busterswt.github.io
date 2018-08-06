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

To support DPDK, the `VT-d` and/or `VT-x` extensions must be enabled in the BIOS.

# Configuring OVS

Over time, some of the popular operating systems have made configuring DPDK and DPDK-enabled Open vSwitch a much easier task. This includes providing packages such as `dpdk` and `openvswitch-switch-dpdk` that take the burden off the operator to compile support for DPDK into Open vSwitch and also figure out how to make persistent DPDK and hugepage configurations. Upcoming patches to OpenStack-Ansible should include support for installing and configuring DPDK-enabled Open vSwitch as well as configuring the host to support DPDK. OpenStack Neutron will also be configured to support such functionality. For now, these instructions assume that OVS has been installed, hugepages have been configured, and everything is ready to go.

While iterating on these patches, I ran into an issue when adding a physical interface to an Open vSwitch bridge using the following command:

```
# ovs-vsctl add-port br-provider dpdk-p0 -- set Interface dpdk-p0 type=dpdk options:dpdk-devargs=0000:03:00.0
ovs-vsctl: Error detected while setting up 'dpdk-p0': Error attaching device '0000:03:00.0' to DPDK.  See ovs-vswitchd log for details.
```

The command above creates a port named `dpdk-p0` of type `dpdk`, associates it with the NIC port at PCI address `0000:03:00.0`, and connects it to the bridge named `br-provider`. In this case, `br-provider` is used as the OpenStack Neutron provider bridge and is connected to the integration bridge just like a normal Open vSwitch deployment (sans DPDK).

The `ovs-vsctl show` output further demonstrates the error:

```
    Bridge br-provider
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-provider
            Interface br-provider
                type: internal
        Port "dpdk-p0"
            Interface "dpdk-p0"
                type: dpdk
                options: {dpdk-devargs="0000:03:00.0"}
                error: "Error attaching device '0000:03:00.0' to DPDK"
        Port phy-br-provider
            Interface phy-br-provider
                type: patch
                options: {peer=int-br-provider}
    ovs_version: "2.9.0"
```



# Summary

The instructions in this guide may not apply only to network interface cards, but also other methods of PCI passthrough using GPUs and other hardware. In my testing, I was able to spin up VMs with the OpenStack API and have them connected to the integration bridge using `dpdkvhostuserclient` ports. I successfully verified connectivity to the VMs from an upstream router. I look forward to kicking the tires on this configuration and performing some benchmarking to see just how much more performance can be squeezed out of the network. If you have any suggestions or feedback, I'm all ears! Hit me up on Twitter at @jimmdenton. 

