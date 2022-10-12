---
title: "[OpenStack] Migrating from LinuxBridge to OVN"
layout: post
date: 2022-08-31
image: /assets/images/2022-08-31-migrating-lxb-to-ovn/walk-away.gif
headerImage: true
tag:
- ovn
- neutron
- openstack
- linuxbridge
category: blog
blog: true
author: jamesdenton
description: "An in-place migration from ML2/LinuxBridge to ML2/OVN"
---

Migrating from one Neutron mechanism driver to another, especially in a production environment, is not a decision one takes on without giving much thought. In many cases, the process involves migrating to a "Greenfield" environment, or a new environment that is stood up running the same or similar operating system and cloud service software but configured in a new way, then migrating entire workloads in a weekend (or more). To say this process is tedious is an understatement.
<!--more-->

Brave individuals have sometimes taken to in-place migrations. In fact, my first OpenStack Summit presentation involved migrating from ML2/OVS to ML2/LXB in-place due to issues with Open vSwitch stability and performance in the early days. Since then, I have been involved with multiple OVS->LXB and LXB->OVS migrations, as well as LXB->OVN. 

## Overview

Since performing the initial migration(s) in the lab, I've decided to better document the process here so you, the reader, can see what's involved and determine if this is the right move for your environment. I'm running OpenStack-Ansible Wallaby, so the steps may need to be extrapolated for environments that involve a more 'manual' process of modifying configurations.

The environment here consists of five nodes:

- 3x controller
- 2x compute

The original plugin/driver is ML2/LinuxBridge with multiple Neutron resources:

- 2x routers
- 2x provider (vlan) networks
- 3x tenant (vxlan) networks

```
root@infra1:~# openstack router list
+--------------------------------------+---------+--------+-------+----------------------------------+
| ID                                   | Name    | Status | State | Project                          |
+--------------------------------------+---------+--------+-------+----------------------------------+
| cee5e805-ecf9-456b-87be-d60f155c8fd8 | rtr-web | ACTIVE | UP    | d1ae5313d10c411fa772e8fa697a6aeb |
| d5052734-53e8-4a58-9fbd-2b76ec138af6 | rtr-db  | ACTIVE | UP    | d1ae5313d10c411fa772e8fa697a6aeb |
+--------------------------------------+---------+--------+-------+----------------------------------+
root@infra1:~# openstack network list
+--------------------------------------+----------------------------------------------------+--------------------------------------+
| ID                                   | Name                                               | Subnets                              |
+--------------------------------------+----------------------------------------------------+--------------------------------------+
| 12a0ab09-d130-4e69-9aa2-c28c66509b02 | db                                                 | 37ae585e-1c48-4aff-98de-dad4f9502428 |
| 282e63e3-5120-4396-a63d-0186e5e96466 | app                                                | d6974d0e-685e-4cfd-ba06-4335c2834788 |
| 3fb2d48e-8c71-4bca-92ce-f64a4c932338 | vlan200                                            | 6e960212-2104-4aea-b51f-686d2b1190d7 |
| 9e151884-67a5-4905-b157-f08f1b3b0040 | HA network tenant d1ae5313d10c411fa772e8fa697a6aeb | 5db20ee1-1d8d-42fe-9724-301fee8c6f43 |
| ab3f0f85-a509-406a-8dca-5db13fbcb48b | web                                                | 90d2e2fe-2301-47b8-b31c-bb6dc7264acb |
| dddfdce8-a8fd-4802-a01c-261b92043488 | vlan100                                            | 6799e6c1-5b66-4894-81b6-6dc698d43462 |
+--------------------------------------+----------------------------------------------------+--------------------------------------+
root@infra1:~# openstack subnet list
+--------------------------------------+---------------------------------------------------+--------------------------------------+------------------+
| ID                                   | Name                                              | Network                              | Subnet           |
+--------------------------------------+---------------------------------------------------+--------------------------------------+------------------+
| 37ae585e-1c48-4aff-98de-dad4f9502428 | db                                                | 12a0ab09-d130-4e69-9aa2-c28c66509b02 | 192.168.55.0/24  |
| 5db20ee1-1d8d-42fe-9724-301fee8c6f43 | HA subnet tenant d1ae5313d10c411fa772e8fa697a6aeb | 9e151884-67a5-4905-b157-f08f1b3b0040 | 169.254.192.0/18 |
| 6799e6c1-5b66-4894-81b6-6dc698d43462 | vlan100                                           | dddfdce8-a8fd-4802-a01c-261b92043488 | 192.168.100.0/24 |
| 6e960212-2104-4aea-b51f-686d2b1190d7 | vlan200                                           | 3fb2d48e-8c71-4bca-92ce-f64a4c932338 | 192.168.200.0/24 |
| 90d2e2fe-2301-47b8-b31c-bb6dc7264acb | web                                               | ab3f0f85-a509-406a-8dca-5db13fbcb48b | 10.5.0.0/24      |
| d6974d0e-685e-4cfd-ba06-4335c2834788 | app                                               | 282e63e3-5120-4396-a63d-0186e5e96466 | 172.25.0.0/24    |
+--------------------------------------+---------------------------------------------------+--------------------------------------+------------------+
```

Six virtual machine instances were deployed across two compute nodes:

```
root@infra1:~# openstack server list
+--------------------------------------+---------+--------+-----------------------------------+--------------+--------+
| ID                                   | Name    | Status | Networks                          | Image        | Flavor |
+--------------------------------------+---------+--------+-----------------------------------+--------------+--------+
| b3a33fb1-98dc-4cf9-99c3-53d5352310e5 | vm-db2  | ACTIVE | db=192.168.55.223                 | cirros-0.5.2 | 1-1-1  |
| e2ea6e2a-aa47-4f44-b285-1b727ad4f709 | vm-db1  | ACTIVE | db=192.168.100.215, 192.168.55.21 | cirros-0.5.2 | 1-1-1  |
| 916052d7-a5f7-4e4a-87a0-7249eef45801 | vm-app2 | ACTIVE | app=172.25.0.250                  | cirros-0.5.2 | 1-1-1  |
| dd3046a3-128a-4585-8ffd-54c11b516052 | vm-app1 | ACTIVE | app=172.25.0.50                   | cirros-0.5.2 | 1-1-1  |
| 7e1af764-a034-4ef2-9695-ca19838812e5 | vm-web1 | ACTIVE | web=10.5.0.121, 192.168.100.90    | cirros-0.5.2 | 1-1-1  |
| dbb98201-52fc-420d-bf6e-5a40fad74327 | vm-web2 | ACTIVE | web=10.5.0.162                    | cirros-0.5.2 | 1-1-1  |
+--------------------------------------+---------+--------+-----------------------------------+--------------+--------+
```

```
root@compute1:~# virsh list
 Id   Name                State
-----------------------------------
 1    instance-00000006   running
 2    instance-0000000c   running
 3    instance-00000012   running

root@compute2:~# virsh list
 Id   Name                State
-----------------------------------
 1    instance-00000009   running
 2    instance-0000000f   running
 3    instance-00000015   running
```

## Inspections

Before conducting the migration, I performed a series of tests to ensure the following was successful:

#### ICMP to all instances from the DHCP namespace(s)

```
root@infra1:~# ip netns exec qdhcp-12a0ab09-d130-4e69-9aa2-c28c66509b02 ping 192.168.55.223 -c2
PING 192.168.55.223 (192.168.55.223) 56(84) bytes of data.
64 bytes from 192.168.55.223: icmp_seq=1 ttl=64 time=13.3 ms
64 bytes from 192.168.55.223: icmp_seq=2 ttl=64 time=1.20 ms

--- 192.168.55.223 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.204/7.239/13.275/6.035 ms
root@infra1:~# ip netns exec qdhcp-12a0ab09-d130-4e69-9aa2-c28c66509b02 ping 192.168.55.21 -c2
PING 192.168.55.21 (192.168.55.21) 56(84) bytes of data.
64 bytes from 192.168.55.21: icmp_seq=1 ttl=64 time=1.61 ms
64 bytes from 192.168.55.21: icmp_seq=2 ttl=64 time=1.19 ms

--- 192.168.55.21 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.193/1.401/1.610/0.208 ms

root@infra1:~# ip netns exec qdhcp-282e63e3-5120-4396-a63d-0186e5e96466 ping 172.25.0.250 -c2
PING 172.25.0.250 (172.25.0.250) 56(84) bytes of data.
64 bytes from 172.25.0.250: icmp_seq=1 ttl=64 time=1.69 ms
64 bytes from 172.25.0.250: icmp_seq=2 ttl=64 time=1.34 ms

--- 172.25.0.250 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.341/1.516/1.691/0.175 ms

root@infra1:~# ip netns exec qdhcp-282e63e3-5120-4396-a63d-0186e5e96466 ping 172.25.0.50 -c2
PING 172.25.0.50 (172.25.0.50) 56(84) bytes of data.
64 bytes from 172.25.0.50: icmp_seq=1 ttl=64 time=1.27 ms
64 bytes from 172.25.0.50: icmp_seq=2 ttl=64 time=1.24 ms

--- 172.25.0.50 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.241/1.255/1.270/0.014 ms

root@infra1:~# ip netns exec qdhcp-ab3f0f85-a509-406a-8dca-5db13fbcb48b ping 10.5.0.121 -c2
PING 10.5.0.121 (10.5.0.121) 56(84) bytes of data.
64 bytes from 10.5.0.121: icmp_seq=1 ttl=64 time=1.94 ms
64 bytes from 10.5.0.121: icmp_seq=2 ttl=64 time=1.44 ms

--- 10.5.0.121 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.437/1.687/1.937/0.250 ms
root@infra1:~# ip netns exec qdhcp-ab3f0f85-a509-406a-8dca-5db13fbcb48b ping 10.5.0.162 -c2
PING 10.5.0.162 (10.5.0.162) 56(84) bytes of data.
64 bytes from 10.5.0.162: icmp_seq=1 ttl=64 time=1.71 ms
64 bytes from 10.5.0.162: icmp_seq=2 ttl=64 time=1.61 ms

--- 10.5.0.162 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.608/1.656/1.705/0.048 ms

```

#### SSH to all instances from the DHCP namespace(s)

```
root@infra1:~# ip netns exec qdhcp-ab3f0f85-a509-406a-8dca-5db13fbcb48b ssh cirros@10.5.0.162 uptime
The authenticity of host '10.5.0.162 (10.5.0.162)' can't be established.
ECDSA key fingerprint is SHA256:NAb9iUzaNKhRptbCLQj/ROZ1vJKisSlFM2amR/s/1Dk.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.5.0.162' (ECDSA) to the list of known hosts.
cirros@10.5.0.162's password:
 14:15:52 up 20 min,  0 users,  load average: 0.00, 0.00, 0.00

root@infra1:~#  ip netns exec qdhcp-282e63e3-5120-4396-a63d-0186e5e96466 ssh cirros@172.25.0.50 uptime
The authenticity of host '172.25.0.50 (172.25.0.50)' can't be established.
ECDSA key fingerprint is SHA256:GMBDGbQ1g1JiyqCTH/kIlrzaojtAXoCGCG/J8BdxEKA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.25.0.50' (ECDSA) to the list of known hosts.
cirros@172.25.0.50's password:
 14:16:26 up 18 min,  0 users,  load average: 0.00, 0.00, 0.00

root@infra1:~# ip netns exec qdhcp-12a0ab09-d130-4e69-9aa2-c28c66509b02 ssh cirros@192.168.55.223 uptime
The authenticity of host '192.168.55.223 (192.168.55.223)' can't be established.
ECDSA key fingerprint is SHA256:WRuu37KvrvU16c7cgF3f4EbA+U9oWMVTY59r/X7rRaA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.55.223' (ECDSA) to the list of known hosts.
cirros@192.168.55.223's password:
 14:16:51 up 13 min,  0 users,  load average: 0.00, 0.00, 0.00
```

#### ICMP between instances

DB2->DB1

```
$ hostname
vm-db2
$ ping 192.168.55.21 -c2
PING 192.168.55.21 (192.168.55.21): 56 data bytes
64 bytes from 192.168.55.21: seq=0 ttl=64 time=1.481 ms
64 bytes from 192.168.55.21: seq=1 ttl=64 time=1.868 ms

--- 192.168.55.21 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 1.481/1.674/1.868 ms
```

WEB1 -> WEB2 and WEB1 -> APP2

```
root@infra1:~# ip netns exec qdhcp-ab3f0f85-a509-406a-8dca-5db13fbcb48b ssh cirros@10.5.0.121
cirros@10.5.0.121's password:
$ hostname
vm-web1
$ ping 10.5.0.162 -c2
PING 10.5.0.162 (10.5.0.162): 56 data bytes
64 bytes from 10.5.0.162: seq=0 ttl=64 time=2.099 ms
64 bytes from 10.5.0.162: seq=1 ttl=64 time=1.880 ms

--- 10.5.0.162 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 1.880/1.989/2.099 ms
$ ping 172.25.0.50 -c2
PING 172.25.0.50 (172.25.0.50): 56 data bytes
64 bytes from 172.25.0.50: seq=0 ttl=63 time=9.040 ms
64 bytes from 172.25.0.50: seq=1 ttl=63 time=2.553 ms

--- 172.25.0.50 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 2.553/5.796/9.040 ms
```

#### Connectivity to floating IP

WEB1->DB1 via FLOAT

```
root@infra1:~# ip netns exec qdhcp-ab3f0f85-a509-406a-8dca-5db13fbcb48b ssh cirros@10.5.0.121
cirros@10.5.0.121's password:
$ hostname
vm-web1
$ ping 192.168.100.215 -c2
PING 192.168.100.215 (192.168.100.215): 56 data bytes
64 bytes from 192.168.100.215: seq=0 ttl=62 time=11.783 ms
64 bytes from 192.168.100.215: seq=1 ttl=62 time=10.835 ms

--- 192.168.100.215 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 10.835/11.309/11.783 ms
```

## Pre-Flight

Before starting the migration, there are a few config changes that can be staged. Please note that this entire process will result in downtime, and is probably not well suited to any sort of "rollback" without serious testing beforehand.

I like to live dangerously.

First, modify the `/etc/openstack_deploy/user_variables.yml` file to include some overrides:

```
neutron_plugin_type: ml2.ovn
neutron_plugin_base:
  - neutron.services.ovn_l3.plugin.OVNL3RouterPlugin
  - qos
neutron_ml2_drivers_type: "geneve,vxlan,vlan,flat"
```

The `qos` plugin may be required if not already enabled in your environment. Previous testing showed that the Neutron API server would not start without it. YMMV.

Next, 	Update the `openstack_inventory.json` inventory file to remove members of L3, DHCP, LinuxBridge and Metadata groups (this will have to be done by hand).

#### BEFORE

```
"neutron_dhcp_agent": {
        "children": [],
        "hosts": [
            "infra1",
            "infra2",
            "infra3"
        ]
    },
    "neutron_l3_agent": {
        "children": [],
        "hosts": [
            "infra1",
            "infra2",
            "infra3"
        ]
    },
    "neutron_linuxbridge_agent": {
        "children": [],
        "hosts": [
            "compute1",
            "compute2",
            "infra1",
            "infra2",
            "infra3"
        ]
    },
    "neutron_metadata_agent": {
        "children": [],
        "hosts": [
            "infra1",
            "infra2",
            "infra3"
        ]
    }
```

#### AFTER

```
"neutron_dhcp_agent": {
        "children": [],
        "hosts": [
        ]
    },
    "neutron_l3_agent": {
        "children": [],
        "hosts": [
        ]
    },
    "neutron_linuxbridge_agent": {
        "children": [],
        "hosts": [
        ]
    },
    "neutron_metadata_agent": {
        "children": [],
        "hosts": [
        ]
    }
```

Then, update the `/etc/openstack_deploy/group_vars/network_hosts` file to add an OVS-related override:

```
openstack_host_specific_kernel_modules:
  - name: "openvswitch"
    pattern: "CONFIG_OPENVSWITCH"
```

Modify the ```/etc/openstack_deploy/env.d/neutron.yml``` file to update Neutron-related group memberships:

```
---
component_skel:
  neutron_ovn_controller:
    belongs_to:
      - neutron_all
  neutron_ovn_northd:
    belongs_to:
      - neutron_all
 
container_skel:
  neutron_agents_container:
    contains: {}
  neutron_ovn_northd_container:
    belongs_to:
      - network_containers
    contains:
      - neutron_ovn_northd
    properties:
      is_metal: true
  neutron_server_container:
    belongs_to:
      - network_containers
    contains:
      - neutron_server
      - opendaylight
    properties:
      is_metal: true
```

Also, modify the ```/etc/openstack_deploy/env.d/nova.yml``` file to update Nova-related group memberships:

```
---
container_skel:
  nova_api_container:
    belongs_to:
      - compute-infra_containers
      - os-infra_containers
    contains:
      - nova_api_metadata
      - nova_api_os_compute
      - nova_conductor
      - nova_scheduler
      - nova_console
    properties:
      is_metal: true
  nova_compute_container:
    belongs_to:
      - compute_containers
      - kvm-compute_containers
      - lxd-compute_containers
      - qemu-compute_containers
    contains:
      - neutron_ovn_controller
      - nova_compute
    properties:
      is_metal: true
```

The network definitions in `openstack_user_config.yml` will need to be updated to reflect changes to support OVN. In this environment there are two bridges: `br-vlan` and `br-flat`. I am taking the opportunity to rename `br-vlan` to `br-ex` to better match upstream documentation. Also, `host_bind_override` is really no good in an OVS-based deployment, we should use `network_interface` instead. 

#### BEFORE

```
    - network:
        container_bridge: "br-vxlan"
        container_type: "veth"
        container_interface: "eth10"
        ip_from_q: "tunnel"
        type: "vxlan"
        range: "1:1000"
        net_name: "vxlan"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-vlan"
        container_type: "veth"
        container_interface: "eth11"
        type: "vlan"
        range: "1:1"
        net_name: "vlan"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-flat"
        container_type: "veth"
        container_interface: "eth12"
        host_bind_override: "veth2"
        type: "flat"
        net_name: "flat"
        group_binds:
          - neutron_linuxbridge_agent
          - utility_all
```

#### AFTER

```
    - network:
        container_bridge: "br-vxlan"
        container_type: "veth"
        container_interface: "eth10"
        ip_from_q: "tunnel"
        type: "geneve"
        range: "1:1000"
        net_name: "geneve"
        group_binds:
          - neutron_ovn_controller
    - network:
        container_bridge: "br-ex"
        container_type: "veth"
        container_interface: "eth11"
        type: "vlan"
        range: "1:1"
        net_name: "vlan"
        group_binds:
          - neutron_ovn_controller
    - network:
        container_bridge: "br-flat"
        container_type: "veth"
        container_interface: "eth12"
        network_interface: "veth2"
        type: "flat"
        net_name: "flat"
        group_binds:
          - neutron_ovn_controller
          - utility_all
```

Once those changes are made, notate and stop all running VMs:

```
root@infra1:~# openstack server list --all | grep ACTIVE
| b3a33fb1-98dc-4cf9-99c3-53d5352310e5 | vm-db2  | ACTIVE | db=192.168.55.223                 | cirros-0.5.2 | 1-1-1  |
| e2ea6e2a-aa47-4f44-b285-1b727ad4f709 | vm-db1  | ACTIVE | db=192.168.100.215, 192.168.55.21 | cirros-0.5.2 | 1-1-1  |
| 916052d7-a5f7-4e4a-87a0-7249eef45801 | vm-app2 | ACTIVE | app=172.25.0.250                  | cirros-0.5.2 | 1-1-1  |
| dd3046a3-128a-4585-8ffd-54c11b516052 | vm-app1 | ACTIVE | app=172.25.0.50                   | cirros-0.5.2 | 1-1-1  |
| 7e1af764-a034-4ef2-9695-ca19838812e5 | vm-web1 | ACTIVE | web=10.5.0.121, 192.168.100.90    | cirros-0.5.2 | 1-1-1  |
| dbb98201-52fc-420d-bf6e-5a40fad74327 | vm-web2 | ACTIVE | web=10.5.0.162                    | cirros-0.5.2 | 1-1-1  |


root@infra1:~# for i in $(openstack server list --all | grep ACTIVE | awk {'print $2'}); do openstack server stop $i; done
```

## Lift Off

Now that everything is staged, it's time to kick off the changes. 

**STOP** and **DISABLE** existing Neutron agents on network and compute hosts:

```
cd /opt/openstack-ansible/playbooks
ansible network_hosts,compute_hosts -m shell -a 'systemctl stop neutron-linuxbridge-agent'
ansible network_hosts,compute_hosts -m shell -a 'systemctl stop neutron-l3-agent'
ansible network_hosts,compute_hosts -m shell -a 'systemctl stop neutron-dhcp-agent'
ansible network_hosts,compute_hosts -m shell -a 'systemctl stop neutron-metadata-agent'
 
ansible network_hosts,compute_hosts -m shell -a 'systemctl disable neutron-linuxbridge-agent'
ansible network_hosts,compute_hosts -m shell -a 'systemctl disable neutron-l3-agent'
ansible network_hosts,compute_hosts -m shell -a 'systemctl disable neutron-dhcp-agent'
ansible network_hosts,compute_hosts -m shell -a 'systemctl disable neutron-metadata-agent'
```

Delete the Neutron-managed network namespaces (qdhcp,qrouter) from controller and compute hosts (repeat as necessary):

```
ssh infra1; 
for i in $(ip netns | grep 'qdhcp\|qrouter' | awk {'print $1'}); do ip netns delete $i; done; 
exit
```

Delete all 'brq' bridges and 'tap' interfaces from controller and compute hosts (repeat as necessary):

```
ssh infra1;
for i in $(ip -br link show | grep brq | awk {'print $1'}); do ip link delete $i; done
for i in $(ip -br link show | grep tap | awk {'print $1'} | sed 's/@.*//'); do ip link delete $i; done
exit;
```

Run the playbooks:

```
cd /opt/openstack-ansible/playbooks
openstack-ansible os-nova-install.yml
openstack-ansible os-neutron-install.yml
```

## Turbulance

After the playbooks have executed, you should expect to have Open vSwitch installed where needed and if configured correctly, you may even have the physical interfaces connected (via `network_interface`).

Check the agent list -- the L3, DHCP, and LXB agents should be down and can be deleted. Metering is TBD:

```
root@infra1:~# openstack network agent list
+--------------------------------------+------------------------------+----------+-------------------+-------+-------+----------------------------+
| ID                                   | Agent Type                   | Host     | Availability Zone | Alive | State | Binary                     |
+--------------------------------------+------------------------------+----------+-------------------+-------+-------+----------------------------+
| 002dd54a-7637-4989-b217-10cf79d6b7f2 | L3 agent                     | infra3   | nova              | XXX   | UP    | neutron-l3-agent           |
| 06ba670f-d560-43aa-b1e7-be60d5914551 | Metering agent               | infra2   | None              | :-)   | UP    | neutron-metering-agent     |
| 1d3bf53f-dcc5-453c-893e-05b053dda55f | Metering agent               | infra3   | None              | :-)   | UP    | neutron-metering-agent     |
| 4353a993-d512-4d42-a554-eccdd4ceeaf8 | Metadata agent               | infra2   | None              | XXX   | UP    | neutron-metadata-agent     |
| 491f1be3-407d-48dd-b8ed-364f2b90c6cb | DHCP agent                   | infra1   | nova              | XXX   | UP    | neutron-dhcp-agent         |
| 55a04c8e-f54e-4e32-81e7-b12c1e2e1c3f | Metadata agent               | infra1   | None              | XXX   | UP    | neutron-metadata-agent     |
| 581954d1-25c8-4b63-a82e-5792250f8b58 | L3 agent                     | infra2   | nova              | XXX   | UP    | neutron-l3-agent           |
| 70d614fa-d3f4-4be9-8fb7-a37eb76d5e38 | DHCP agent                   | infra2   | nova              | XXX   | UP    | neutron-dhcp-agent         |
| 76a84bb3-84b1-4ba9-b2bb-00f4d2776b6d | Linux bridge agent           | infra2   | None              | XXX   | UP    | neutron-linuxbridge-agent  |
| 8153be7b-82d5-4077-b5df-b7e414756220 | Linux bridge agent           | compute2 | None              | XXX   | UP    | neutron-linuxbridge-agent  |
| 9ed0c376-44c1-4150-b7ab-23138cee7430 | Linux bridge agent           | infra3   | None              | XXX   | UP    | neutron-linuxbridge-agent  |
| a7d803df-191e-413c-bafc-23049c7732e0 | Linux bridge agent           | compute1 | None              | XXX   | UP    | neutron-linuxbridge-agent  |
| bb9986d4-5d44-4040-8490-a1a5af1feb33 | Metadata agent               | infra3   | None              | XXX   | UP    | neutron-metadata-agent     |
| d04290ee-1215-42b2-af34-3ce84eada471 | Metering agent               | infra1   | None              | :-)   | UP    | neutron-metering-agent     |
| d1bd340d-8da8-4915-8eba-f7078d08e9ed | Linux bridge agent           | infra1   | None              | XXX   | UP    | neutron-linuxbridge-agent  |
| eb868890-7b6d-41e3-8fbd-54730963bca7 | DHCP agent                   | infra3   | nova              | XXX   | UP    | neutron-dhcp-agent         |
| f13952a5-ba28-4767-ad3c-b72fe6c0db6a | L3 agent                     | infra1   | nova              | XXX   | UP    | neutron-l3-agent           |
| fc536b52-a35c-4523-885d-0708759445e0 | OVN Controller Gateway agent | compute1 |                   | :-)   | UP    | ovn-controller             |
| d60d8a20-d977-4352-a886-c7b5ef477446 | OVN Controller Gateway agent | compute2 |                   | :-)   | UP    | ovn-controller             |
| c3c7ff97-998c-5adb-ac2a-75c930724959 | OVN Metadata agent           | compute2 |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| f20d28dd-83c7-5589-8f9f-37a4f974996d | OVN Metadata agent           | compute1 |                   | :-)   | UP    | neutron-ovn-metadata-agent |
+--------------------------------------+------------------------------+----------+-------------------+-------+-------+----------------------------+
```

**DELETE** the now-stale agents:

```
root@infra1:~# for i in $(openstack network agent list | grep XXX | awk {'print $2'}); do openstack network agent delete $i; done

root@infra1:~# openstack network agent list
+--------------------------------------+------------------------------+----------+-------------------+-------+-------+----------------------------+
| ID                                   | Agent Type                   | Host     | Availability Zone | Alive | State | Binary                     |
+--------------------------------------+------------------------------+----------+-------------------+-------+-------+----------------------------+
| 06ba670f-d560-43aa-b1e7-be60d5914551 | Metering agent               | infra2   | None              | :-)   | UP    | neutron-metering-agent     |
| 1d3bf53f-dcc5-453c-893e-05b053dda55f | Metering agent               | infra3   | None              | :-)   | UP    | neutron-metering-agent     |
| d04290ee-1215-42b2-af34-3ce84eada471 | Metering agent               | infra1   | None              | :-)   | UP    | neutron-metering-agent     |
| fc536b52-a35c-4523-885d-0708759445e0 | OVN Controller Gateway agent | compute1 |                   | :-)   | UP    | ovn-controller             |
| d60d8a20-d977-4352-a886-c7b5ef477446 | OVN Controller Gateway agent | compute2 |                   | :-)   | UP    | ovn-controller             |
| c3c7ff97-998c-5adb-ac2a-75c930724959 | OVN Metadata agent           | compute2 |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| f20d28dd-83c7-5589-8f9f-37a4f974996d | OVN Metadata agent           | compute1 |                   | :-)   | UP    | neutron-ovn-metadata-agent |
+--------------------------------------+------------------------------+----------+-------------------+-------+-------+----------------------------+
```

Check the OVN DBs using the local server IP - the northbound database is likely empty, while the southbound database should be populated:

```
root@infra1:~# ovn-nbctl --db=tcp:10.0.236.100:6641 show
root@infra1:~# ovn-sbctl --db=tcp:10.0.236.100:6642 show
Chassis "d60d8a20-d977-4352-a886-c7b5ef477446"
    hostname: compute2
    Encap vxlan
        ip: "10.0.240.121"
        options: {csum="true"}
    Encap geneve
        ip: "10.0.240.121"
        options: {csum="true"}
Chassis "fc536b52-a35c-4523-885d-0708759445e0"
    hostname: compute1
    Encap vxlan
        ip: "10.0.240.120"
        options: {csum="true"}
    Encap geneve
        ip: "10.0.240.120"
        options: {csum="true"}
```

An empty northbound database is the result of a lack of sync between OVN and Neutron, and can be resolved by running the `neutron-ovn-db-sync-util` command in `repair` mode:

```
/openstack/venvs/neutron-23.4.1.dev3/bin/neutron-ovn-db-sync-util \
--config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini \
--ovn-neutron_sync_mode repair
```

#### EXAMPLE

```
Example:
root@infra1:~# /openstack/venvs/neutron-23.4.1.dev3/bin/neutron-ovn-db-sync-util \
> --config-file /etc/neutron/neutron.conf \
> --config-file /etc/neutron/plugins/ml2/ml2_conf.ini \
> --ovn-neutron_sync_mode repair
/openstack/venvs/neutron-23.4.1.dev3/lib/python3.8/site-packages/sqlalchemy/orm/relationships.py:1994: SAWarning: Setting backref / back_populates on relationship QosNetworkPolicyBinding.port to refer to viewonly relationship Port.qos_network_policy_binding should include sync_backref=False set on the QosNetworkPolicyBinding.port relationship.  (this warning may be suppressed after 10 occurrences)
  util.warn_limited(
/openstack/venvs/neutron-23.4.1.dev3/lib/python3.8/site-packages/sqlalchemy/orm/relationships.py:1994: SAWarning: Setting backref / back_populates on relationship Tag.standard_attr to refer to viewonly relationship StandardAttribute.tags should include sync_backref=False set on the Tag.standard_attr relationship.  (this warning may be suppressed after 10 occurrences)
  util.warn_limited(
root@infra1:~# echo $?
0
```

A successful run should result in logical switch, ports, floating IPs, etc. being populated in the northbound DB:

```
root@infra1:~# ovn-nbctl --db=tcp:10.0.236.100:6641 show
switch 44717724-6a70-4ee7-b0ab-143bdcb12c79 (neutron-12a0ab09-d130-4e69-9aa2-c28c66509b02) (aka db)
    port f3e92114-f005-4029-81e3-65f1d60e8862
        addresses: ["fa:16:3e:8c:1b:8f 192.168.55.2", "unknown"]
    port 196d3e0e-295d-4317-a04c-e9d950160e61
        addresses: ["fa:16:3e:b6:b1:7f 192.168.55.223"]
    port c762d350-417c-4a3b-b40c-d595dafcc368
        type: localport
        addresses: ["fa:16:3e:e4:21:7c 192.168.55.5"]
    port b0c5e704-fe40-4538-9232-94a091b7adb7
        addresses: ["fa:16:3e:6c:6d:57 192.168.55.21"]
    port 60c8459c-be90-44b0-8007-56cfa995da4f
        addresses: ["fa:16:3e:19:0c:14 192.168.55.4", "unknown"]
    port 76dd53d9-7eac-4bc6-92f9-48cf975235b5
        type: router
        router-port: lrp-76dd53d9-7eac-4bc6-92f9-48cf975235b5
    port 2837c67c-c7c0-44dd-be82-1192226cb7b8
        addresses: ["fa:16:3e:dc:07:6f 192.168.55.3", "unknown"]
switch 38efee60-da0c-4ed8-ad21-e76ce12a4cb3 (neutron-282e63e3-5120-4396-a63d-0186e5e96466) (aka app)
    port c8caf3c7-ac86-4cb5-85eb-12e88f3713eb
        addresses: ["fa:16:3e:73:70:29 172.25.0.50"]
    port ae7b0df8-4343-448b-af68-5f3afd78e869
        type: localport
        addresses: ["fa:16:3e:56:b9:6a 172.25.0.5"]
    port a448543b-fe5c-4aaf-aef4-cdcd6421e84b
        addresses: ["fa:16:3e:56:b7:dc 172.25.0.2", "unknown"]
    port 4626c5a9-f578-4849-8fb3-93700f3ddb06
        addresses: ["fa:16:3e:60:8c:2d 172.25.0.250"]
    port cf2644b6-abc4-42e7-bbdb-0e204f261446
        addresses: ["fa:16:3e:aa:42:c8 172.25.0.3", "unknown"]
    port 8dcc107d-272f-4ced-b601-3090171ce01c
        addresses: ["fa:16:3e:51:6c:b8 172.25.0.4", "unknown"]
    port 9b4eb252-e69e-43e1-8585-8b79c986d07c
        type: router
        router-port: lrp-9b4eb252-e69e-43e1-8585-8b79c986d07c
switch f56b65e0-8638-4e0d-baeb-2194ec8dacac (neutron-3fb2d48e-8c71-4bca-92ce-f64a4c932338) (aka vlan200)
    port 07e29662-6e52-4ccb-b2de-361a888e633c
        addresses: ["fa:16:3e:c7:9b:37 192.168.200.4", "unknown"]
    port c2bbb3e0-2c65-45d7-b8f6-573e665dfc6e
        addresses: ["fa:16:3e:e4:69:4f 192.168.200.2", "unknown"]
    port d493513f-8b81-4e01-9dfd-89b43f2fa3f5
        addresses: ["fa:16:3e:da:44:d6 192.168.200.3", "unknown"]
    port c3d08e61-41a9-4495-83f1-6720cf798c75
        type: localport
        addresses: ["fa:16:3e:05:61:e9 192.168.200.5"]
    port provnet-ccfefb4d-0da0-4138-ac74-be1934eca9d7
        type: localnet
        tag: 200
        addresses: ["unknown"]
switch 08c14ec5-c809-467d-92e4-a9dc5092217e (neutron-dddfdce8-a8fd-4802-a01c-261b92043488) (aka vlan100)
    port ee9592f0-d028-4941-ad02-77385cd371aa
        type: router
        router-port: lrp-ee9592f0-d028-4941-ad02-77385cd371aa
    port d4478a35-0406-46b9-bab9-17df99e1e44c
        addresses: ["fa:16:3e:a4:71:eb 192.168.100.2", "unknown"]
    port 7484fd6e-c82c-4679-aa5b-a7f7b6ef5f9a
        type: localport
        addresses: ["fa:16:3e:57:3c:8f 192.168.100.5"]
    port 555e54dd-4edc-4286-84d1-d639cc7fb143
        addresses: ["fa:16:3e:6b:80:b5 192.168.100.4", "unknown"]
    port provnet-fc4d896e-9eb8-4a73-a363-223a5dc81ec5
        type: localnet
        tag: 100
        addresses: ["unknown"]
    port 10062270-348a-473a-8ed0-f551cfacfce5
        type: router
        router-port: lrp-10062270-348a-473a-8ed0-f551cfacfce5
    port ae75e43e-c4e6-4972-8917-59f5779b3d5c
        addresses: ["fa:16:3e:07:86:b5 192.168.100.3", "unknown"]
switch 74bd93f5-0434-4c64-8b65-1a44d4370bef (neutron-9e151884-67a5-4905-b157-f08f1b3b0040) (aka HA network tenant d1ae5313d10c411fa772e8fa697a6aeb)
    port 15a2cb60-d85f-4ae2-b867-4621c4e66b72 (aka HA port tenant d1ae5313d10c411fa772e8fa697a6aeb)
        type: router
        router-port: lrp-15a2cb60-d85f-4ae2-b867-4621c4e66b72
    port cdb9739f-9b11-453e-b1c2-3bfbb8bad187 (aka HA port tenant d1ae5313d10c411fa772e8fa697a6aeb)
        type: router
        router-port: lrp-cdb9739f-9b11-453e-b1c2-3bfbb8bad187
    port 9634c76e-e309-40cf-b701-1bcf38b4bde4
        type: localport
        addresses: ["fa:16:3e:2e:78:c3"]
    port 106ede6e-1f6f-4c17-a478-9e58045da88b (aka HA port tenant d1ae5313d10c411fa772e8fa697a6aeb)
        type: router
        router-port: lrp-106ede6e-1f6f-4c17-a478-9e58045da88b
    port e1918430-42ca-40dc-aa59-5c54934e121c (aka HA port tenant d1ae5313d10c411fa772e8fa697a6aeb)
        type: router
        router-port: lrp-e1918430-42ca-40dc-aa59-5c54934e121c
    port 850149e1-8f9c-4c64-8b47-90df032a8d65 (aka HA port tenant d1ae5313d10c411fa772e8fa697a6aeb)
        type: router
        router-port: lrp-850149e1-8f9c-4c64-8b47-90df032a8d65
    port 3670eea6-7adf-42d9-b524-cd438cf51a09 (aka HA port tenant d1ae5313d10c411fa772e8fa697a6aeb)
        type: router
        router-port: lrp-3670eea6-7adf-42d9-b524-cd438cf51a09
switch de0dac56-f19e-45b9-b7fb-ee12ccb2fea4 (neutron-ab3f0f85-a509-406a-8dca-5db13fbcb48b) (aka web)
    port 09380ac6-cfcf-4969-b704-4f0de6433f89
        type: router
        router-port: lrp-09380ac6-cfcf-4969-b704-4f0de6433f89
    port 45f94f9a-2a3d-489e-9b64-23002a1d495c
        addresses: ["fa:16:3e:b6:02:c7 10.5.0.3", "unknown"]
    port 37511b88-ee9a-4be2-bb46-fff22d01d5af
        addresses: ["fa:16:3e:6d:3a:91 10.5.0.162"]
    port 53a7a6c2-d350-4e98-91e3-9c3df6ebc3e2
        addresses: ["fa:16:3e:a2:cd:b8 10.5.0.4", "unknown"]
    port cf4dc1f3-0774-496a-87e1-b9954cb90320
        type: localport
        addresses: ["fa:16:3e:31:2b:9b 10.5.0.5"]
    port 637e2a67-c198-4a1d-b836-55757227eb39
        addresses: ["fa:16:3e:b5:d2:44 10.5.0.121"]
    port 7a94ca01-8e5a-4248-b56d-8343e3a15fe8
        addresses: ["fa:16:3e:10:c9:68 10.5.0.2", "unknown"]
router 5e9befda-2983-44a9-ab68-195858ca89f8 (neutron-d5052734-53e8-4a58-9fbd-2b76ec138af6) (aka rtr-db)
    port lrp-cdb9739f-9b11-453e-b1c2-3bfbb8bad187
        mac: "fa:16:3e:82:19:af"
        networks: ["169.254.194.128/18"]
    port lrp-850149e1-8f9c-4c64-8b47-90df032a8d65
        mac: "fa:16:3e:c9:6c:7d"
        networks: ["169.254.193.130/18"]
    port lrp-15a2cb60-d85f-4ae2-b867-4621c4e66b72
        mac: "fa:16:3e:8f:03:97"
        networks: ["169.254.195.150/18"]
    port lrp-76dd53d9-7eac-4bc6-92f9-48cf975235b5
        mac: "fa:16:3e:f0:d2:09"
        networks: ["192.168.55.1/24"]
    port lrp-10062270-348a-473a-8ed0-f551cfacfce5
        mac: "fa:16:3e:64:41:cb"
        networks: ["192.168.100.235/24"]
        gateway chassis: [d60d8a20-d977-4352-a886-c7b5ef477446 fc536b52-a35c-4523-885d-0708759445e0]
    nat 6c32224c-1d97-44a4-abb8-184bea546880
        external ip: "192.168.100.235"
        logical ip: "169.254.192.0/18"
        type: "snat"
    nat 917d1234-ebb7-4578-a8c0-5355302e5aab
        external ip: "192.168.100.215"
        logical ip: "192.168.55.21"
        type: "dnat_and_snat"
    nat df8a97a6-0954-4ccc-a742-26e32c493974
        external ip: "192.168.100.235"
        logical ip: "192.168.55.0/24"
        type: "snat"
    nat e0d236dc-63e0-4c51-9451-588a3ac5c051
        external ip: "192.168.100.235"
        logical ip: "169.254.192.0/18"
        type: "snat"
    nat f0ee4069-7d38-47f2-ad48-9d6b393aa773
        external ip: "192.168.100.235"
        logical ip: "169.254.192.0/18"
        type: "snat"
router dfde8f9a-c539-48ec-82ce-a3fda47c7a86 (neutron-cee5e805-ecf9-456b-87be-d60f155c8fd8) (aka rtr-web)
    port lrp-e1918430-42ca-40dc-aa59-5c54934e121c
        mac: "fa:16:3e:0a:cc:de"
        networks: ["169.254.193.111/18"]
    port lrp-ee9592f0-d028-4941-ad02-77385cd371aa
        mac: "fa:16:3e:60:72:8a"
        networks: ["192.168.100.202/24"]
        gateway chassis: [fc536b52-a35c-4523-885d-0708759445e0 d60d8a20-d977-4352-a886-c7b5ef477446]
    port lrp-09380ac6-cfcf-4969-b704-4f0de6433f89
        mac: "fa:16:3e:90:36:8f"
        networks: ["10.5.0.1/24"]
    port lrp-3670eea6-7adf-42d9-b524-cd438cf51a09
        mac: "fa:16:3e:3a:fe:53"
        networks: ["169.254.195.230/18"]
    port lrp-106ede6e-1f6f-4c17-a478-9e58045da88b
        mac: "fa:16:3e:be:b5:54"
        networks: ["169.254.194.238/18"]
    port lrp-9b4eb252-e69e-43e1-8585-8b79c986d07c
        mac: "fa:16:3e:70:e3:46"
        networks: ["172.25.0.1/24"]
    nat 02530272-13cf-4690-aad9-4160943b7418
        external ip: "192.168.100.202"
        logical ip: "172.25.0.0/24"
        type: "snat"
    nat 4149ca39-b9f2-4e35-8be7-9c0d3a343bf7
        external ip: "192.168.100.202"
        logical ip: "10.5.0.0/24"
        type: "snat"
    nat bc084986-d1a6-4877-810a-ca39fc01d064
        external ip: "192.168.100.202"
        logical ip: "169.254.192.0/18"
        type: "snat"
    nat d83383c4-91f4-47ca-9fac-9d1cfb56ed01
        external ip: "192.168.100.202"
        logical ip: "169.254.192.0/18"
        type: "snat"
    nat dddaa10e-6a24-4133-8efa-612517247c88
        external ip: "192.168.100.202"
        logical ip: "169.254.192.0/18"
        type: "snat"
    nat ef26f8a2-b61b-431f-a288-3fa6c6b6488c
        external ip: "192.168.100.90"
        logical ip: "10.5.0.121"
        type: "dnat_and_snat"
```

## Approach

One of the last steps of this process is one of the trickiest: Neutron ports must be updated to reflect a `vif_type` of `ovs` rather than `bridge`. Unfortunately, this is not an API-driven change but one that must be done within the database itself.

The following command can be used:

```
use neutron;
update ml2_port_bindings set vif_type='ovs' where vif_type='bridge';
```

#### EXAMPLE

```
MariaDB [neutron]> select * from ml2_port_bindings where vif_type='bridge';
+--------------------------------------+----------+----------+-----------+---------+---------------------------------------------+--------+
| port_id                              | host     | vif_type | vnic_type | profile | vif_details                                 | status |
+--------------------------------------+----------+----------+-----------+---------+---------------------------------------------+--------+
| 07e29662-6e52-4ccb-b2de-361a888e633c | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 09380ac6-cfcf-4969-b704-4f0de6433f89 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 106ede6e-1f6f-4c17-a478-9e58045da88b | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 15a2cb60-d85f-4ae2-b867-4621c4e66b72 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 196d3e0e-295d-4317-a04c-e9d950160e61 | compute2 | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 2837c67c-c7c0-44dd-be82-1192226cb7b8 | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 3670eea6-7adf-42d9-b524-cd438cf51a09 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 37511b88-ee9a-4be2-bb46-fff22d01d5af | compute1 | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 45f94f9a-2a3d-489e-9b64-23002a1d495c | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 4626c5a9-f578-4849-8fb3-93700f3ddb06 | compute2 | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 53a7a6c2-d350-4e98-91e3-9c3df6ebc3e2 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 555e54dd-4edc-4286-84d1-d639cc7fb143 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 60c8459c-be90-44b0-8007-56cfa995da4f | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 637e2a67-c198-4a1d-b836-55757227eb39 | compute2 | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 76dd53d9-7eac-4bc6-92f9-48cf975235b5 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 7a94ca01-8e5a-4248-b56d-8343e3a15fe8 | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 850149e1-8f9c-4c64-8b47-90df032a8d65 | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 8dcc107d-272f-4ced-b601-3090171ce01c | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| 9b4eb252-e69e-43e1-8585-8b79c986d07c | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| a448543b-fe5c-4aaf-aef4-cdcd6421e84b | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| ae75e43e-c4e6-4972-8917-59f5779b3d5c | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| b0c5e704-fe40-4538-9232-94a091b7adb7 | compute1 | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| c2bbb3e0-2c65-45d7-b8f6-573e665dfc6e | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| c8caf3c7-ac86-4cb5-85eb-12e88f3713eb | compute1 | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| cdb9739f-9b11-453e-b1c2-3bfbb8bad187 | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| cf2644b6-abc4-42e7-bbdb-0e204f261446 | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| d4478a35-0406-46b9-bab9-17df99e1e44c | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| d493513f-8b81-4e01-9dfd-89b43f2fa3f5 | infra2   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| e1918430-42ca-40dc-aa59-5c54934e121c | infra1   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
| f3e92114-f005-4029-81e3-65f1d60e8862 | infra3   | bridge   | normal    |         | {"connectivity": "l2", "port_filter": true} | ACTIVE |
+--------------------------------------+----------+----------+-----------+---------+---------------------------------------------+--------+

MariaDB [neutron]> update ml2_port_bindings set vif_type='ovs' where vif_type='bridge';
Query OK, 30 rows affected (0.026 sec)
Rows matched: 30  Changed: 30  Warnings: 0
```

Also, according to the OVN [manpage](https://www.ovn.org/support/dist-docs/ovn-controller.8.html), VXLAN networks are only supported for gateway nodes and not traffic between hypervisors:

```
external_ids:ovn-encap-type
   The encapsulation type that a chassis should use to  con‐
   nect  to  this  node. Multiple encapsulation types may be
   specified with a comma-separated list. Each listed encap‐
   sulation type will be paired with ovn-encap-ip.

   Supported  tunnel  types  for  connecting hypervisors are
   geneve and stt. Gateways may use geneve, vxlan, or stt.
```

So, the DB can also be munged to make this change, too:

```
MariaDB [neutron]> select * from networksegments;
+--------------------------------------+--------------------------------------+--------------+------------------+-----------------+------------+---------------+------------------+------+
| id                                   | network_id                           | network_type | physical_network | segmentation_id | is_dynamic | segment_index | standard_attr_id | name |
+--------------------------------------+--------------------------------------+--------------+------------------+-----------------+------------+---------------+------------------+------+
| 84334795-0a9c-46dc-bb45-abd858a787ae | 282e63e3-5120-4396-a63d-0186e5e96466 | vxlan        | NULL             |             942 |          0 |             0 |               27 | NULL |
| 8c866571-b041-426c-9ddd-5f126fd694e3 | ab3f0f85-a509-406a-8dca-5db13fbcb48b | vxlan        | NULL             |             230 |          0 |             0 |               21 | NULL |
| 949f5d71-fd93-45e3-9895-ad2415541e89 | 9e151884-67a5-4905-b157-f08f1b3b0040 | vxlan        | NULL             |              10 |          0 |             0 |              144 | NULL |
| b4f4131d-f988-49ef-9440-361d747af8eb | 12a0ab09-d130-4e69-9aa2-c28c66509b02 | vxlan        | NULL             |             665 |          0 |             0 |               33 | NULL |
| ccfefb4d-0da0-4138-ac74-be1934eca9d7 | 3fb2d48e-8c71-4bca-92ce-f64a4c932338 | vlan         | vlan             |             200 |          0 |             0 |              126 | NULL |
| fc4d896e-9eb8-4a73-a363-223a5dc81ec5 | dddfdce8-a8fd-4802-a01c-261b92043488 | vlan         | vlan             |             100 |          0 |             0 |              120 | NULL |
+--------------------------------------+--------------------------------------+--------------+------------------+-----------------+------------+---------------+------------------+------+

MariaDB [neutron]> update networksegments set network_type='geneve' where network_type='vxlan';
Query OK, 4 rows affected (0.008 sec)
Rows matched: 4  Changed: 4  Warnings: 0
```

## Soft landing

At this point, all of the tough changes have been made and it's time to try out our new toy.

```
root@infra1:~# openstack server start vm-web1
get() takes 1 positional argument but 2 were given
```

Uh oh.

```
root@infra1:~# openstack server start vm-web1
```

That's better.

Checking the console of the VM demonstrates proper DHCP and Metadata connectivity:

```
root@infra1:~# openstack console log show vm-web1
...
Starting network: udhcpc: started, v1.29.3
udhcpc: sending discover
udhcpc: sending select for 10.5.0.121
udhcpc: lease of 10.5.0.121 obtained, lease time 43200
...
checking http://169.254.169.254/2009-04-04/instance-id
successful after 1/20 tries: up 3.04. iid=i-00000009
...
```

Let's try spinning up the others:

```
root@infra1:~# openstack server start vm-web2
root@infra1:~# openstack server start vm-app1
get() takes 1 positional argument but 2 were given
root@infra1:~# openstack server start vm-app1
root@infra1:~# openstack server start vm-app2
get() takes 1 positional argument but 2 were given
root@infra1:~# openstack server start vm-app2
root@infra1:~# openstack server start vm-db1
get() takes 1 positional argument but 2 were given
root@infra1:~# openstack server start vm-db1
root@infra1:~# openstack server start vm-db2
Networking client is experiencing an unauthorized exception. (HTTP 400) (Request-ID: req-3a3348c9-39fb-49f2-84b2-f2b3c3e9e466)
root@infra1:~# openstack server start vm-db2
```

It looks like the first VMs of a given network are the only ones to complain, which could be related to stale cache or something else that gets resolved automatically. Without looking at the API logs, it's hard to say.

```
root@infra1:~# openstack server list
+--------------------------------------+---------+--------+-----------------------------------+--------------+--------+
| ID                                   | Name    | Status | Networks                          | Image        | Flavor |
+--------------------------------------+---------+--------+-----------------------------------+--------------+--------+
| b3a33fb1-98dc-4cf9-99c3-53d5352310e5 | vm-db2  | ACTIVE | db=192.168.55.223                 | cirros-0.5.2 | 1-1-1  |
| e2ea6e2a-aa47-4f44-b285-1b727ad4f709 | vm-db1  | ACTIVE | db=192.168.100.215, 192.168.55.21 | cirros-0.5.2 | 1-1-1  |
| 916052d7-a5f7-4e4a-87a0-7249eef45801 | vm-app2 | ACTIVE | app=172.25.0.250                  | cirros-0.5.2 | 1-1-1  |
| dd3046a3-128a-4585-8ffd-54c11b516052 | vm-app1 | ACTIVE | app=172.25.0.50                   | cirros-0.5.2 | 1-1-1  |
| 7e1af764-a034-4ef2-9695-ca19838812e5 | vm-web1 | ACTIVE | web=10.5.0.121, 192.168.100.90    | cirros-0.5.2 | 1-1-1  |
| dbb98201-52fc-420d-bf6e-5a40fad74327 | vm-web2 | ACTIVE | web=10.5.0.162                    | cirros-0.5.2 | 1-1-1  |
+--------------------------------------+---------+--------+-----------------------------------+--------------+--------+
```

## Inspection

The moment of truth is here, but performing checks from DHCP namespaces that no longer exist will be tricky. Fortunately, an `ovmmeta` namespace exists on each node that is connected to their respective network. Unfortunately, the namespace is only connected to the *local* bridge and cannot communicate across hosts. 

The following example demonstrates connectivity from the `ovnmeta` namespace to vm-db1, and from within vm-db1 to vm-db2 (across hosts):

```
root@compute1:~# ip netns exec ovnmeta-12a0ab09-d130-4e69-9aa2-c28c66509b02 ssh cirros@192.168.55.21
cirros@192.168.55.21's password:
$ hostname
vm-db1
$ ping 192.168.55.223 -c2
PING 192.168.55.223 (192.168.55.223): 56 data bytes
64 bytes from 192.168.55.223: seq=0 ttl=64 time=4.595 ms
64 bytes from 192.168.55.223: seq=1 ttl=64 time=2.770 ms

$ ssh cirros@192.168.55.223

Host '192.168.55.223' is not in the trusted hosts file.
(ecdsa-sha2-nistp256 fingerprint sha1!! ae:44:c1:5c:da:13:06:05:56:22:76:0d:0c:82:1e:84:bf:e8:2d:9c)
Do you want to continue connecting? (y/n) y
cirros@192.168.55.223's password:
$ hostname
vm-db2
```

Here we ping from vm-web2 to vm-app1 and vm-app2

```
$ hostname
vm-web2
$ ping 172.25.0.50 -c2
PING 172.25.0.50 (172.25.0.50): 56 data bytes
64 bytes from 172.25.0.50: seq=0 ttl=63 time=1.578 ms
64 bytes from 172.25.0.50: seq=1 ttl=63 time=1.125 ms

--- 172.25.0.50 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 1.125/1.351/1.578 ms

$ ping 172.25.0.250 -c2
PING 172.25.0.250 (172.25.0.250): 56 data bytes
64 bytes from 172.25.0.250: seq=0 ttl=63 time=5.781 ms
64 bytes from 172.25.0.250: seq=1 ttl=63 time=3.353 ms

--- 172.25.0.250 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 3.353/4.567/5.781 ms
```

Lastly, we can see floating IP traffic between vm-web1 -> vm-db2 works as well:

```
$ hostname
vm-web1
$ ping 192.168.100.215 -c2
PING 192.168.100.215 (192.168.100.215): 56 data bytes
64 bytes from 192.168.100.215: seq=0 ttl=62 time=14.153 ms
64 bytes from 192.168.100.215: seq=1 ttl=62 time=4.775 ms

--- 192.168.100.215 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 4.775/9.464/14.153 ms
```

## Summary

Being able to perform in-place migrations and upgrades is important, especially when the resources don't exist to perform a "lift-n-shift" type of migration. When looking to perform an in-place migration, my suggestion is to always ***TEST TEST TEST*** in a similarly-configured lab environment to work out all kninks and potential unknowns. Make configuration and database backups, and be prepared to lose instances in a worst-case scenario.  

---
If you have some thoughts or comments on this post, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
