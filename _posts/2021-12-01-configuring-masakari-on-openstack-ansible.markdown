---
title: "Configuring Masakari (Instance HA) on OpenStack-Ansible"
layout: post
date: 2021-12-01
image: assets/images/2021-12-01-configuring-masakari-on-openstack-ansible/impact-of-downtime.png
headerImage: true
tag:
- openstack
- virtualization
- ha
- masakari
category: blog
blog: true
author: jamesdenton
description: "Configuring Masakari (Instance HA) on OpenStack-Ansible"
---

Providing high availability of cloud resources, whether it be networking or virtual machines, is a topic that comes up often in my corner of the world. I'd heard of Instance-HA in some Red Hat circles, and only recently learned the OpenStack Masakari project was the one to provide that functionality.<!--more-->

That said, I'd like to kick the tires on it, and what better place to start than my own OpenStack-based lab running OpenStack-Ansible (Xena).

## Getting Started

OpenStack Masakari provides high availability of instances by performing the following actions:

- Host evacuation
- Instance restart

**Host evacuation** is a feature provided by a monitor known as **masakari-hostmonitor**. With this feature, all instances are evacuated from a node that is considered *DOWN*. There are some requirements for this feature, such as shared storage, and some other considerations that must be made, including the need to fence offline hosts to ensure the evacuation is successful.

**Instance restart** is provided by a monitor known as **masakari-instancemonitor**. With this feature, an instance is restarted should its process on the compute node die. 

There are additional monitors provided by Masakari, including:

- masakari-processmonitor
- masakari-introspectivemonitor

The **masakari-processmonitor** monitor can be used to monitor other processes and services on the host to ensure they are running consistently. Processes and services can be added to the `process_list.yaml` file found at `/etc/masakarimonitors/process_list.yaml` on ***compute*** nodes or other nodes running monitoring agents. Monitored services can be modified using the `masakari_monitors_process_overrides` override in OpenStack-Ansible.

Lastly, the **masakari-introspectivemonitor** monitor can be used to detect system-level failure events via the qemu-qa protocol. Not much has been written about this particular monitor as of yet.


### Host Evacuation

With Masakari, compute nodes are grouped into ***failover segments***. In the event of a compute node failure, that node’s instances are moved onto another compute node within the *same* segment. Failover segments are not to be confused with other logical groups of computes, such as availability zones or aggregates, but represent a similar concept.

The destination node is determined by the recovery method configured for the affected segment. There are four methods:

- reserved_host
- auto
- rh_priority
- auto_priority

The **reserved_host** recovery method relocates instances to a subset of *non-active* nodes. Because these nodes are not active and are typically resourced adequately for failover duty of similarly-equipped active nodes, there is a guarantee that sufficient resources will exist on a reserved node to accommodate migrated instances.

The **auto** recovery method relocates instances to ***any*** available node in the same segment. Because all the nodes are active, however, there is no guarantee that sufficient resources will exist on the destination node to accommodate migrated instances.

The **rh_priority** recovery method attempts to evacuate instances using the reserved_host method first, and falls back to the auto method should the reserved_host method fail.

The **auto_priority** recovery method attempts to evacuate instances using the auto method first, and falls back to the reserved_host method should the auto method fail.

Host evacuation requires shared storage and some method of fencing nodes, likely provided by Pacemaker/STONITH and access to the OOB management network. Given these requirements and an incomplete implementation within OpenStack-Ansible at this time, I'll skip this demonstration.

### Instance Restart

The enabling of the instance restart feature is done on a *per-instance* basis using the `HA_Enabled=True` property. Once Masakari has been deployed, an agent on the compute node will detect instance failure and (hopefully) restart the instance according to policy.

## Configuring and Deploying

In an OpenStack-Ansible environment, managing the inventory and group membership is key to deploying.

To enable Masakari, simply add the following to the `openstack_user_config.yml` file:

```
masakari-infra_hosts: *infrastructure_hosts
masakari-monitor_hosts: *compute_hosts
```

Keep in mind, those aliases will only work if you've defined them in your environment, like so:

```
infrastructure_hosts: &infrastructure_hosts
  lab-infra01:
    ip: 10.20.0.30
    no_containers: true
  lab-infra02:
    ip: 10.20.0.22
    no_containers: true
  lab-infra03:
    ip: 10.20.0.23
    no_containers: true

compute_hosts: &compute_hosts
  lab-compute01:
    ip: 10.20.0.31
  lab-compute02:
    ip: 10.20.0.32
```

Then, execute the following playbooks:

- haproxy-install.yml
- os-masakari-install.yml

Once installed, you should notice a few new services across your infrastructure and compute nodes, including:

#### Infra:

- masakari-api.service                               
- masakari-engine.service  

#### Compute:
                          
- masakari-hostmonitor.service                       
- masakari-instancemonitor.service                   
- masakari-introspectiveinstancemonitor.service      
- masakari-processmonitor.service  

## Testing Instance Restart

To test automatic instance restart, I first spun up an instance with the following command:

```
openstack server create --image "Ubuntu Server 20.04 Focal" --boot-from-volume 20 --flavor m1.small --network LAN --security-group SSH --key-name imac-rsa masakari-vm1

+--------------------------------------+---------------+---------+--------------------+--------------------------+-----------+
| ID                                   | Name          | Status  | Networks           | Image                    | Flavor    |
+--------------------------------------+---------------+---------+--------------------+--------------------------+-----------+
| 5101ed69-00c9-4956-b7fd-7f2256c23474 | masakari-vm1  | ACTIVE  | LAN=192.168.2.188  | N/A (booted from volume) | m1.small  |
+--------------------------------------+---------------+---------+--------------------+--------------------------+-----------+
```      

Then, I set the `HA_Enabled` property to `True`:

```
openstack server set --property HA_Enabled=True masakari-vm1
```

To simulate an unexpected failure, I killed the instance on the compute node:

```
root@lab-compute01:~# pgrep -f guest=instance-00000450
154259
root@lab-compute01:~# pkill -f -9 guest=instance-00000450
```

At the same time, we can see the following events taking place in the log:

```
Dec  1 16:36:28 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:28.660 151091 INFO masakarimonitors.instancemonitor.libvirt_handler.callback [-] Libvirt Event: type=VM, hostname=lab-compute01, uuid=5101ed69-00c9-4956-b7fd-7f2256c23474, time=2021-12-01 16:36:28.660446, event_id=LIFECYCLE, detail=STOPPED_FAILED)
Dec  1 16:36:28 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:28.661 151091 INFO masakarimonitors.ha.masakari [-] Send a notification. {'notification': {'type': 'VM', 'hostname': 'lab-compute01', 'generated_time': datetime.datetime(2021, 12, 1, 16, 36, 28, 660446), 'payload': {'event': 'LIFECYCLE', 'instance_uuid': '5101ed69-00c9-4956-b7fd-7f2256c23474', 'vir_domain_event': 'STOPPED_FAILED'}}}
Dec  1 16:36:29 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:29.910 151091 INFO masakarimonitors.ha.masakari [-] Response: openstack.instance_ha.v1.notification.Notification(type=VM, hostname=lab-compute01, generated_time=2021-12-01T16:36:28.660446, payload={'event': 'LIFECYCLE', 'instance_uuid': '5101ed69-00c9-4956-b7fd-7f2256c23474', 'vir_domain_event': 'STOPPED_FAILED'}, id=15, notification_uuid=bc7eb253-b5fc-4feb-92fd-46e494772f0d, source_host_uuid=6d39c8c7-9d8f-4faf-a7e7-bbcd7dd5d79d, status=new, created_at=2021-12-01T16:36:29.000000, updated_at=None, location=Munch({'cloud': '10.20.0.11', 'region_name': 'RegionOne', 'zone': None, 'project': Munch({'id': '36de0c24e456401d8df6ffaff42224d0', 'name': None, 'domain_id': None, 'domain_name': None})}))
Dec  1 16:36:43 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:43.372 151091 INFO masakarimonitors.instancemonitor.libvirt_handler.callback [-] Libvirt Event: type=VM, hostname=lab-compute01, uuid=5101ed69-00c9-4956-b7fd-7f2256c23474, time=2021-12-01 16:36:43.371886, event_id=REBOOT, detail=UNKNOWN)
Dec  1 16:36:43 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:43.373 151091 INFO masakarimonitors.ha.masakari [-] Send a notification. {'notification': {'type': 'VM', 'hostname': 'lab-compute01', 'generated_time': datetime.datetime(2021, 12, 1, 16, 36, 43, 371886), 'payload': {'event': 'REBOOT', 'instance_uuid': '5101ed69-00c9-4956-b7fd-7f2256c23474', 'vir_domain_event': 'UNKNOWN'}}}
Dec  1 16:36:43 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:43.383 151091 INFO masakarimonitors.instancemonitor.libvirt_handler.callback [-] Libvirt Event: type=VM, hostname=lab-compute01, uuid=5101ed69-00c9-4956-b7fd-7f2256c23474, time=2021-12-01 16:36:43.383194, event_id=REBOOT, detail=UNKNOWN)
Dec  1 16:36:43 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:43.384 151091 INFO masakarimonitors.ha.masakari [-] Send a notification. {'notification': {'type': 'VM', 'hostname': 'lab-compute01', 'generated_time': datetime.datetime(2021, 12, 1, 16, 36, 43, 383194), 'payload': {'event': 'REBOOT', 'instance_uuid': '5101ed69-00c9-4956-b7fd-7f2256c23474', 'vir_domain_event': 'UNKNOWN'}}}
Dec  1 16:36:44 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:44.818 151091 INFO masakarimonitors.ha.masakari [-] Response: openstack.instance_ha.v1.notification.Notification(type=VM, hostname=lab-compute01, generated_time=2021-12-01T16:36:43.371886, payload={'event': 'REBOOT', 'instance_uuid': '5101ed69-00c9-4956-b7fd-7f2256c23474', 'vir_domain_event': 'UNKNOWN'}, id=18, notification_uuid=7a08ef3c-ce4a-48a3-ba1a-05d8890c4b9f, source_host_uuid=6d39c8c7-9d8f-4faf-a7e7-bbcd7dd5d79d, status=new, created_at=2021-12-01T16:36:44.000000, updated_at=None, location=Munch({'cloud': '10.20.0.11', 'region_name': 'RegionOne', 'zone': None, 'project': Munch({'id': '36de0c24e456401d8df6ffaff42224d0', 'name': None, 'domain_id': None, 'domain_name': None})}))
Dec  1 16:36:44 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 16:36:44.918 151091 INFO masakarimonitors.ha.masakari [-] Stop retrying to send a notification because same notification have been already sent.
```

A look at the process list shows a new PID:

```
root@lab-compute01:~# pgrep -f guest=instance-00000450
168720
```

Finally, a simultaneous ping test shows the ping fail and subseqeuently recover once the instance has been restarted:

```
root@lab-infra01:~# ping 192.168.2.188
PING 192.168.2.188 (192.168.2.188) 56(84) bytes of data.
64 bytes from 192.168.2.188: icmp_seq=1 ttl=64 time=2.22 ms
64 bytes from 192.168.2.188: icmp_seq=2 ttl=64 time=1.06 ms
64 bytes from 192.168.2.188: icmp_seq=3 ttl=64 time=0.891 ms
64 bytes from 192.168.2.188: icmp_seq=4 ttl=64 time=0.909 ms
64 bytes from 192.168.2.188: icmp_seq=5 ttl=64 time=0.933 ms
64 bytes from 192.168.2.188: icmp_seq=6 ttl=64 time=0.813 ms
64 bytes from 192.168.2.188: icmp_seq=7 ttl=64 time=0.978 ms
64 bytes from 192.168.2.188: icmp_seq=8 ttl=64 time=0.967 ms
64 bytes from 192.168.2.188: icmp_seq=9 ttl=64 time=0.884 ms
64 bytes from 192.168.2.188: icmp_seq=10 ttl=64 time=0.969 ms
64 bytes from 192.168.2.188: icmp_seq=11 ttl=64 time=0.937 ms
64 bytes from 192.168.2.188: icmp_seq=12 ttl=64 time=0.876 ms
...
64 bytes from 192.168.2.188: icmp_seq=40 ttl=64 time=1.97 ms
64 bytes from 192.168.2.188: icmp_seq=41 ttl=64 time=0.886 ms
64 bytes from 192.168.2.188: icmp_seq=42 ttl=64 time=1.70 ms
64 bytes from 192.168.2.188: icmp_seq=43 ttl=64 time=0.728 ms
```

## Testing Service Restart

A similar test can be used to verify the automatic restart of crucial services, such as libvirtd, nova-compute, and others.

Here, I ungracefully kill the libvirtd process:

```
root@lab-compute01:~# pgrep -f libvirtd
183773
183807
root@lab-compute01:~# killall libvirtd
```

Masakari gets to work restarting the service:

```
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 20:21:56.399 151091 WARNING masakarimonitors.instancemonitor.instance [-] Libvirt Connection Closed Unexpectedly.
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 20:21:56.400 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 20:21:56.400 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 20:21:56.401 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: message repeated 2 times: [ 2021-12-01 20:21:56.401 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed]
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 20:21:56.402 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: message repeated 2 times: [ 2021-12-01 20:21:56.402 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed]
Dec  1 20:21:56 lab-compute01 masakari-instancemonitor[151091]: 2021-12-01 20:21:56.403 151091 WARNING masakarimonitors.instancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-introspectiveinstancemonitor[151131]: 2021-12-01 20:21:56.460 151131 WARNING masakarimonitors.introspectiveinstancemonitor.instance [-] Libvirt Connection Closed Unexpectedly.
Dec  1 20:21:56 lab-compute01 masakari-introspectiveinstancemonitor[151131]: 2021-12-01 20:21:56.461 151131 WARNING masakarimonitors.introspectiveinstancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-introspectiveinstancemonitor[151131]: 2021-12-01 20:21:56.461 151131 WARNING masakarimonitors.introspectiveinstancemonitor.instance [-] Error from libvirt : internal error: client socket is closed
Dec  1 20:21:56 lab-compute01 masakari-processmonitor[184385]: 2021-12-01 20:21:56.937 184385 WARNING masakarimonitors.processmonitor.process_handler.handle_process [-] Process '/usr/sbin/libvirtd' is not found.
Dec  1 20:21:57 lab-compute01 masakari-processmonitor[184385]: 2021-12-01 20:21:57.046 184385 INFO masakarimonitors.processmonitor.process_handler.handle_process [-] Restart of process with executing command: systemctl restart libvirtd
```

A look at the process list shows a new PID:

```
root@lab-compute01:~# pgrep -f libvirtd
184026
184063
```

## Summary

Automated *anything* can be a delicate balance of risk and reward. I'm glad to have had some time looking at OpenStack Masakari, and while the instance restart functionality is great, I'm really looking forward to helping implement the host evacation capabilities within OpenStack-Ansible in the coming <del>weeks</del> <del>months</del> <del>years</del> *sometime*.

---
If you have some thoughts or comments on this article, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
