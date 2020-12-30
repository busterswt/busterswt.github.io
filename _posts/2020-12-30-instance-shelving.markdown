---
title: "OpenStack: Instance Shelving"
layout: post
date: 2020-03-09
image: /assets/images/2020-12-30-instance-shelving/woman-with-a-full-closet.jpg
headerImage: true
tag:
- openstack
- virtualization
- nova
- shelving
category: blog
blog: true
author: jamesdenton
description: "Exploring instance shelving with Nova"
---

My good friend Cody Bunch [documented](https://blog.codybunch.com/2017/09/08/OpenStack-Image-Shelving/) OpenStack instance shelving back in 2017, and I recently revisited the topic for a customer looking to better conserve resources in their busy cloud. They have developers all over the world working around the clock, and a limited set of resources to go around.<!--more-->

In a nutshell, "shelving" an instance allows one to stop an instance and regain the associated resources (i.e. vcpu, ram, disk) without having to completely delete the instance. The compute service creates a snapshot of the instance and uploads it to the Glance image library. The running instance is effectively *deleted* from the compute node, but runtime details such as vCPU(s), memory, disk size, and IP address(es) are retained for eventual unshelving.

Ordinarily, when an instance is shutdown, the resources tied to the instance continue to be reserved. This behavior can result in a compute node being full of non-running instances and unable to be scheduled to! When an instance is *shelved*, however, its resources are freed up and made available. When the instance is unshelved, Nova will subsequently rebuild the instance using the respective snapshot and schedule it where appropriate.

Easy peasy, right?

To demonstrate this behavior in action, I'll be working with a lab environment running Ussuri consisting of a single controller node, two compute nodes, and a Glance image store over NFS.

Take a look at the resources currently consumed on the two compute nodes:

```
🌕OpenStack Lab % openstack host show lab-compute01
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute01 | (total)                          |  32 |    257974 |    3646 |
| lab-compute01 | (used_now)                       |   4 |     10240 |      80 |
| lab-compute01 | (used_max)                       |   4 |      8192 |      80 |
| lab-compute01 | 7a8df96a3c6a47118e60e57aa9ecff54 |   4 |      8192 |      80 |
+---------------+----------------------------------+-----+-----------+---------+
🌕OpenStack Lab % openstack host show lab-compute02
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute02 | (total)                          |  48 |    128796 |     937 |
| lab-compute02 | (used_now)                       |  18 |     21504 |     140 |
| lab-compute02 | (used_max)                       |  18 |     19456 |     150 |
| lab-compute02 | 7a8df96a3c6a47118e60e57aa9ecff54 |  17 |     18432 |     130 |
| lab-compute02 | 36de0c24e456401d8df6ffaff42224d0 |   1 |      1024 |      20 |
+---------------+----------------------------------+-----+-----------+---------+
```
There are two projects with instances spread across both nodes. I will use the `openstack` client to create an instance consuming 2 vCPUs, 4 GB RAM and 40 GB disk:

```
🌕OpenStack Lab % openstack server create --flavor 2-4-40 --image a299a16d-f46c-4e4e-9dc4-515634fc4ac1 --network LAN --key-name imac-rsa --security-group bench save-10
+-------------------------------------+-------------------------------------------------------------------+
| Field                               | Value                                                             |
+-------------------------------------+-------------------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                            |
| OS-EXT-AZ:availability_zone         |                                                                   |
| OS-EXT-SRV-ATTR:host                | None                                                              |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                                              |
| OS-EXT-SRV-ATTR:instance_name       |                                                                   |
| OS-EXT-STS:power_state              | NOSTATE                                                           |
| OS-EXT-STS:task_state               | scheduling                                                        |
| OS-EXT-STS:vm_state                 | building                                                          |
| OS-SRV-USG:launched_at              | None                                                              |
| OS-SRV-USG:terminated_at            | None                                                              |
| accessIPv4                          |                                                                   |
| accessIPv6                          |                                                                   |
| addresses                           |                                                                   |
| adminPass                           | opjQcrG2ffTM                                                      |
| config_drive                        |                                                                   |
| created                             | 2020-12-30T15:25:14Z                                              |
| flavor                              | 2-4-40 (8a7c77fa-98da-4df0-9df4-5227a553dd70)                     |
| hostId                              |                                                                   |
| id                                  | 6495f1fd-f440-4db8-8502-f6c6bd8971ba                              |
| image                               | Ubuntu Server 18.04 Bionic (a299a16d-f46c-4e4e-9dc4-515634fc4ac1) |
| key_name                            | imac-rsa                                                          |
| name                                | save-10                                                           |
| progress                            | 0                                                                 |
| project_id                          | 7a8df96a3c6a47118e60e57aa9ecff54                                  |
| properties                          |                                                                   |
| security_groups                     | name='23ecf413-93fc-42fe-89ef-e0fafaa2f934'                       |
| status                              | BUILD                                                             |
| updated                             | 2020-12-30T15:25:14Z                                              |
| user_id                             | 34f3cf48b24f41c097555c07961f139e                                  |
| volumes_attached                    |                                                                   |
+-------------------------------------+-------------------------------------------------------------------+
```

After a few moments, the instance goes `ACTIVE` and responds to ping:

```
🌕OpenStack Lab % openstack server list
+--------------------------------------+---------+-------------------+-------------------------------+----------------------------+--------------------+
| ID                                   | Name    | Status            | Networks                      | Image                      | Flavor             |
+--------------------------------------+---------+-------------------+-------------------------------+----------------------------+--------------------+
| 6495f1fd-f440-4db8-8502-f6c6bd8971ba | save-10 | ACTIVE            | LAN=192.168.2.209             | Ubuntu Server 18.04 Bionic | 2-4-40             |
+--------------------------------------+---------+-------------------+-------------------------------+----------------------------+--------------------+

➜  ~ ping 192.168.2.209
PING 192.168.2.209 (192.168.2.209): 56 data bytes
64 bytes from 192.168.2.209: icmp_seq=0 ttl=64 time=7.365 ms
64 bytes from 192.168.2.209: icmp_seq=1 ttl=64 time=7.594 ms
64 bytes from 192.168.2.209: icmp_seq=2 ttl=64 time=3.963 ms
64 bytes from 192.168.2.209: icmp_seq=3 ttl=64 time=4.007 ms
64 bytes from 192.168.2.209: icmp_seq=4 ttl=64 time=4.107 ms
64 bytes from 192.168.2.209: icmp_seq=5 ttl=64 time=5.535 ms
...
```

Resource consumption is updated accordingly on compute01:

```
🌕OpenStack Lab % openstack host show lab-compute01
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute01 | (total)                          |  32 |    257974 |    3646 |
| lab-compute01 | (used_now)                       |   6 |     14336 |     120 |
| lab-compute01 | (used_max)                       |   6 |     12288 |     120 |
| lab-compute01 | 7a8df96a3c6a47118e60e57aa9ecff54 |   6 |     12288 |     120 |
+---------------+----------------------------------+-----+-----------+---------+

🌕OpenStack Lab % openstack host show lab-compute02
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute02 | (total)                          |  48 |    128796 |     937 |
| lab-compute02 | (used_now)                       |  18 |     21504 |     140 |
| lab-compute02 | (used_max)                       |  18 |     19456 |     150 |
| lab-compute02 | 7a8df96a3c6a47118e60e57aa9ecff54 |  17 |     18432 |     130 |
| lab-compute02 | 36de0c24e456401d8df6ffaff42224d0 |   1 |      1024 |      20 |
+---------------+----------------------------------+-----+-----------+---------+
```

If we simply shutdown the instance, the instance goes offline but the resources remain tied up:

```
🌕OpenStack Lab % openstack server stop save-10
🌕OpenStack Lab % openstack server show save-10 -c OS-EXT-STS:task_state -c status
+-----------------------+---------+
| Field                 | Value   |
+-----------------------+---------+
| OS-EXT-STS:task_state | None    |
| status                | SHUTOFF |
+-----------------------+---------+
🌕OpenStack Lab % openstack host show lab-compute01
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute01 | (total)                          |  32 |    257974 |    3646 |
| lab-compute01 | (used_now)                       |   6 |     14336 |     120 |
| lab-compute01 | (used_max)                       |   6 |     12288 |     120 |
| lab-compute01 | 7a8df96a3c6a47118e60e57aa9ecff54 |   6 |     12288 |     120 |
+---------------+----------------------------------+-----+-----------+---------+
```

To meet the needs of the user, the instance needs to be shutdown but remain available for re-use. Simply shutting down the instance isn't enough, since the resources are not freed up. This is where shelving comes in.

# Shelving

To shelve the instance, make sure it's in an `ACTIVE` state and issue the `openstack server shelve` command:

```
🌕OpenStack Lab % openstack server shelve save-10
```

No feedback is given. However, the ping will begin to fail and the `task_state` will change accordingly:

```
64 bytes from 192.168.2.209: icmp_seq=22 ttl=64 time=5.311 ms
64 bytes from 192.168.2.209: icmp_seq=23 ttl=64 time=5.429 ms
64 bytes from 192.168.2.209: icmp_seq=24 ttl=64 time=4.238 ms
64 bytes from 192.168.2.209: icmp_seq=25 ttl=64 time=3.949 ms
Request timeout for icmp_seq 26
Request timeout for icmp_seq 27
Request timeout for icmp_seq 28
Request timeout for icmp_seq 29
Request timeout for icmp_seq 30
Request timeout for icmp_seq 31
...

🌕OpenStack Lab % openstack server show save-10 -c OS-EXT-STS:task_state -c status
+-----------------------+--------------------------+
| Field                 | Value                    |
+-----------------------+--------------------------+
| OS-EXT-STS:task_state | shelving_image_uploading |
| status                | ACTIVE                   |
+-----------------------+--------------------------+
```

Once complete, the instance status will change:

```
🌕OpenStack Lab % openstack server show save-10 -c OS-EXT-STS:task_state -c status
+-----------------------+-------------------+
| Field                 | Value             |
+-----------------------+-------------------+
| OS-EXT-STS:task_state | None              |
| status                | SHELVED_OFFLOADED |
+-----------------------+-------------------+
```

If we check the resources again, we will find they have been returned:

```
🌕OpenStack Lab % openstack host show lab-compute01
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute01 | (total)                          |  32 |    257974 |    3646 |
| lab-compute01 | (used_now)                       |   4 |     10240 |      80 |
| lab-compute01 | (used_max)                       |   4 |      8192 |      80 |
| lab-compute01 | 7a8df96a3c6a47118e60e57aa9ecff54 |   4 |      8192 |      80 | 
+---------------+----------------------------------+-----+-----------+---------+
🌕OpenStack Lab % openstack host show lab-compute02
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute02 | (total)                          |  48 |    128796 |     937 |
| lab-compute02 | (used_now)                       |  18 |     21504 |     140 |
| lab-compute02 | (used_max)                       |  18 |     19456 |     150 |
| lab-compute02 | 7a8df96a3c6a47118e60e57aa9ecff54 |  17 |     18432 |     130 |
| lab-compute02 | 36de0c24e456401d8df6ffaff42224d0 |   1 |      1024 |      20 |
+---------------+----------------------------------+-----+-----------+---------+
```

A look at compute01 shows 2 vCPU, 2 GB RAM, and 40 GB disk have been freed up.

Unshelving the instance is as simple as `openstack server unshelve`:

```
🌕OpenStack Lab % openstack server unshelve save-10
```

There is no immediate feedback. The instance will respawn in a location determined by the Nova scheduler. Notice the various state changes as the instance comes online:

```
🌕OpenStack Lab % openstack server show save-10 -c OS-EXT-STS:task_state -c status
+-----------------------+-------------------+
| Field                 | Value             |
+-----------------------+-------------------+
| OS-EXT-STS:task_state | unshelving        |
| status                | SHELVED_OFFLOADED |
+-----------------------+-------------------+
🌕OpenStack Lab % openstack server show save-10 -c OS-EXT-STS:task_state -c status
+-----------------------+-------------------+
| Field                 | Value             |
+-----------------------+-------------------+
| OS-EXT-STS:task_state | spawning          |
| status                | SHELVED_OFFLOADED |
+-----------------------+-------------------+
🌕OpenStack Lab % openstack server show save-10 -c OS-EXT-STS:task_state -c status
+-----------------------+--------+
| Field                 | Value  |
+-----------------------+--------+
| OS-EXT-STS:task_state | None   |
| status                | ACTIVE |
+-----------------------+--------+
```

Once `ACTIVE`, the instance begins pinging again:

```
...
Request timeout for icmp_seq 951
Request timeout for icmp_seq 952
Request timeout for icmp_seq 953
Request timeout for icmp_seq 954
64 bytes from 192.168.2.209: icmp_seq=955 ttl=64 time=9.667 ms
64 bytes from 192.168.2.209: icmp_seq=956 ttl=64 time=8.269 ms
64 bytes from 192.168.2.209: icmp_seq=957 ttl=64 time=8.848 ms
64 bytes from 192.168.2.209: icmp_seq=958 ttl=64 time=10.406 ms
64 bytes from 192.168.2.209: icmp_seq=959 ttl=64 time=7.649 ms
...
```

Now we can see the resources consumed on compute02:

```
Now we see resources claimed on lab-compute02:

🌕OpenStack Lab % openstack host show lab-compute01
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute01 | (total)                          |  32 |    257974 |    3646 |
| lab-compute01 | (used_now)                       |   4 |     10240 |      80 |
| lab-compute01 | (used_max)                       |   4 |      8192 |      80 |
| lab-compute01 | 7a8df96a3c6a47118e60e57aa9ecff54 |   4 |      8192 |      80 |
+---------------+----------------------------------+-----+-----------+---------+
🌕OpenStack Lab % openstack host show lab-compute02
+---------------+----------------------------------+-----+-----------+---------+
| Host          | Project                          | CPU | Memory MB | Disk GB |
+---------------+----------------------------------+-----+-----------+---------+
| lab-compute02 | (total)                          |  48 |    128796 |     937 |
| lab-compute02 | (used_now)                       |  20 |     25600 |     180 |
| lab-compute02 | (used_max)                       |  20 |     23552 |     190 |
| lab-compute02 | 7a8df96a3c6a47118e60e57aa9ecff54 |  19 |     22528 |     170 |
| lab-compute02 | 36de0c24e456401d8df6ffaff42224d0 |   1 |      1024 |      20 |
+---------------+----------------------------------+-----+-----------+---------+
```

# Image Library

Prior to shelving, our image library consisted of a single image:

```
🌕OpenStack Lab % openstack image list
+--------------------------------------+-----------------------------------+--------+
| ID                                   | Name                              | Status |
+--------------------------------------+-----------------------------------+--------+
| a299a16d-f46c-4e4e-9dc4-515634fc4ac1 | Ubuntu Server 18.04 Bionic        | active |
+--------------------------------------+-----------------------------------+--------+
```

After shelving, our snapshot appears in the list:

```
🌕OpenStack Lab % openstack image list
+--------------------------------------+-----------------------------------+--------+
| ID                                   | Name                              | Status |
+--------------------------------------+-----------------------------------+--------+
| a299a16d-f46c-4e4e-9dc4-515634fc4ac1 | Ubuntu Server 18.04 Bionic        | active |
| 5df4680e-5f81-4d3b-aa74-3aece41fdb22 | save-10-shelved                   | active |
+--------------------------------------+-----------------------------------+--------+
```

The details of the snapshot can be seen here:

```
🌕OpenStack Lab % openstack image show save-10-shelved
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | e9226299db2b4d927e0c7e509de52e0e                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| container_format | bare                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| created_at       | 2020-12-30T15:44:52Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| disk_format      | qcow2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| file             | /v2/images/5df4680e-5f81-4d3b-aa74-3aece41fdb22/file                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| id               | 5df4680e-5f81-4d3b-aa74-3aece41fdb22                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| min_disk         | 40                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| min_ram          | 0                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| name             | save-10-shelved                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| owner            | 7a8df96a3c6a47118e60e57aa9ecff54                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| properties       | base_image_ref='a299a16d-f46c-4e4e-9dc4-515634fc4ac1', boot_roles='reader,admin,member', clean_attempts='3', image_location='snapshot', image_state='available', image_type='snapshot', instance_uuid='6495f1fd-f440-4db8-8502-f6c6bd8971ba', locations='[{'url': 'file:///var/lib/glance/images/5df4680e-5f81-4d3b-aa74-3aece41fdb22', 'metadata': {'store': 'file'}}]', os_hash_algo='sha512', os_hash_value='8903476ac71d81cd92898cf71d73c560d9acec2e61a1150f702bf77459416c8f7bde28bc063048a6fe8aab67f80ce3ea0b2d524bd8f145d0a0e49f540e5c1eea', os_hidden='False', owner_project_name='admin', owner_specified.openstack.md5='ed44b9745b8d62bcbbc180b5f36c24bb', owner_specified.openstack.object='images/Ubuntu Server 18.04 Bionic', owner_specified.openstack.sha256='c4517e054c398235aa7a09ddcc1db31cd168077049febcc4292ff77fe1e5eab3', owner_user_name='admin', stores='file', user_id='34f3cf48b24f41c097555c07961f139e' |
| protected        | False                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| schema           | /v2/schemas/image                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| size             | 1155530752                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| status           | active                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| tags             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| updated_at       | 2020-12-30T15:45:15Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| virtual_size     | 42949672960                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| visibility       | private                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Notice the visibility is `private`. It will only be visible to the owning project. A quick test found that I could create additional instances using this snapshot, which may or may not be a security concern. I am not sure if other users in the same project would be able to do the same with this image, however.

Anyhow, this solution may prove useful as users shelve their instances over weekends or overnight to allow resources to be consumed by other users of the cloud. Just be sure to have enough free space to store the snapshots!

If you have some thoughts or comments on this process, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.