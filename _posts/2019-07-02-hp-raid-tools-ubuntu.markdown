---
title: "Installing and Using HP Smart Storage Utilities on Ubuntu 18.04 LTS"
layout: post
date: 2019-07-02
image: /assets/images/2019-07-02-hp-raid-tools-ubuntu/redundancy.png
headerImage: true
tag:
- raid
- openstack-ansible
- hpe
- openstack
- proliant
category: blog
blog: true
author: jamesdenton
description: "Installing and Using HP Smart Storage Utilities on Ubuntu 18.04 LTS"
---

As I branch out of the networking world and into general systems administration duties, I find myself having to learn a lot more about the tools and utilities used to manage said systems. I recently deployed Cinder in my OpenStack-Ansible based homelab, and am attempting to learn and use the tools available to me in a more efficient way.
<!--more-->

My OpenStack cluster consists of the following:

- 1x Infrastructure Node: HP DL360e G8 (4x LFF)
- 1x Compute Node: HP DL380p G8 (12x LFF w/ HP 420i)

The task was simple: Add a new disk to the host to be setup in a RAID 0 and used for Cinder volumes.

# Install the packages

Installing a new drive and preparing it for use in the system can be accomplished in one of two ways:

1. Insert the drive and reboot the host into the Smart Storage Utility where you can configure the drive.
2. Use the HP Smart Storage Utilities for Linux to modify the RAID configuration within the host OS

Option 2 sounded much better. However, many of the posts that walk through this process are outdated and reference APT sources that are no longer available or utilities whose names have changed a few times over the years.

The following instructions will help you, the reader, configure the latest packages for Ubuntu 18.04 LTS (Bionic) to manage your RAID within the host OS.

First, create a new APT source named `hpe.list` at `/etc/apt/sources.list.d/` and add the following:

```
deb http://downloads.linux.hpe.com/SDR/repo/mcp bionic/current non-free
```

Next, update APT:

```
# apt update
```

Finally, install the relevant packages:

```
# apt install ssa ssacli ssaducli
```

# Basic sscli commands

Once the packages have been installed, use the `ssacli ctrl all show config` command to view the current configuration:

```
root@lab-compute01:~# ssacli ctrl all show config

Smart Array P420i in Slot 0 (Embedded)    (sn: 00143802965EB30)


   Gen8 ServBP 12+2 at Port 1I, Box 1, OK

   Port Name: 1I

   Port Name: 2I

   Array A (SATA, Unused Space: 0  MB)

      logicaldrive 1 (7.28 TB, RAID 1+0, OK)

      physicaldrive 1I:1:1 (port 1I:box 1:bay 1, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:2 (port 1I:box 1:bay 2, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:4 (port 1I:box 1:bay 4, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:5 (port 1I:box 1:bay 5, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:7 (port 1I:box 1:bay 7, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:8 (port 1I:box 1:bay 8, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:10 (port 1I:box 1:bay 10, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:11 (port 1I:box 1:bay 11, SATA HDD, 2 TB, OK)

   Enclosure SEP (Vendor ID HP, Model Gen8 ServBP 12+2) 378  (WWID: 50014380303EAAF9, Port: 1I, Box: 1)

   Expander 380  (WWID: 50014380303EAAE0, Port: 1I, Box: 1)

   SEP (Vendor ID PMCSIERA, Model SRCv8x6G) 379  (WWID: 500143802965EB3F)
```

Use the `ssacli ctrl all show status` command to view the current status of the controller:

```
root@lab-compute01:~# ssacli ctrl all show status

Smart Array P420i in Slot 0 (Embedded)
   Controller Status: OK
   Cache Status: OK
   Battery/Capacitor Status: OK
```

I found a recently-updated blog containing additional useful commands at [https://wiki.phoenixlzx.com/page/ssacli/](https://wiki.phoenixlzx.com/page/ssacli/). Check it out!


# Adding a drive

At this point, I inserted the drive into the chassis. Re-running the `ssacli ctrl all show config` command, I could see the controller recognized the drive:

```
root@lab-compute01:~# ssacli ctrl all show config

Smart Array P420i in Slot 0 (Embedded)    (sn: 00143802965EB30)



   Gen8 ServBP 12+2 at Port 1I, Box 1, OK


   Port Name: 1I

   Port Name: 2I

   Array A (SATA, Unused Space: 0  MB)

      logicaldrive 1 (7.28 TB, RAID 1+0, OK)

      physicaldrive 1I:1:1 (port 1I:box 1:bay 1, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:2 (port 1I:box 1:bay 2, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:4 (port 1I:box 1:bay 4, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:5 (port 1I:box 1:bay 5, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:7 (port 1I:box 1:bay 7, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:8 (port 1I:box 1:bay 8, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:10 (port 1I:box 1:bay 10, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:11 (port 1I:box 1:bay 11, SATA HDD, 2 TB, OK)

   Unassigned

      physicaldrive 1I:1:3 (port 1I:box 1:bay 3, SATA HDD, 2 TB, OK)

   Enclosure SEP (Vendor ID HP, Model Gen8 ServBP 12+2) 378  (WWID: 50014380303EAAF9, Port: 1I, Box: 1)

   Expander 380  (WWID: 50014380303EAAE0, Port: 1I, Box: 1)

   SEP (Vendor ID PMCSIERA, Model SRCv8x6G) 379  (WWID: 500143802965EB3F)
```

Notice this here:

```
   Unassigned

      physicaldrive 1I:1:3 (port 1I:box 1:bay 3, SATA HDD, 2 TB, OK)
```

In my system, Bay 3 is located at the bottom left, as shown here:

![Bay 3](/assets/images/2019-07-02-hp-raid-tools-ubuntu/bay3.png)

To create a new RAID 0 logical drive, use `sscli ctrl create` command:

```
# ssacli ctrl slot=0 create type=ld drives=1I:1:3 raid=0
```

Another look at the config should reflect the newly-built logical drive:

```
root@lab-compute01:~# ssacli ctrl all show config
...
   Array A (SATA, Unused Space: 0  MB)

      logicaldrive 1 (7.28 TB, RAID 1+0, OK)

      physicaldrive 1I:1:1 (port 1I:box 1:bay 1, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:2 (port 1I:box 1:bay 2, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:4 (port 1I:box 1:bay 4, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:5 (port 1I:box 1:bay 5, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:7 (port 1I:box 1:bay 7, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:8 (port 1I:box 1:bay 8, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:10 (port 1I:box 1:bay 10, SATA HDD, 2 TB, OK)
      physicaldrive 1I:1:11 (port 1I:box 1:bay 11, SATA HDD, 2 TB, OK)


   Array B (SATA, Unused Space: 0  MB)

      logicaldrive 2 (1.82 TB, RAID 0, OK)

      physicaldrive 1I:1:3 (port 1I:box 1:bay 3, SATA HDD, 2 TB, OK)
...
```

You can then use the `sscli ctrl show` command to see how this drive was mapped within the operating system (where `ld 2` maps to `logicaldrive 2` above):

```
root@lab-compute01:~# ssacli ctrl slot=0 ld 2 show

Smart Array P420i in Slot 0 (Embedded)

   Array B

      Logical Drive: 2
         Size: 1.82 TB
         Fault Tolerance: 0
         Heads: 255
         Sectors Per Track: 32
         Cylinders: 65535
         Strip Size: 256 KB
         Full Stripe Size: 256 KB
         Status: OK
         Caching:  Enabled
         Unique Identifier: 600508B1001CA08B21AD7F9F4DAFDA25
         Disk Name: /dev/sdd
         Mount Points: None
         Logical Drive Label: 058522AB00143802965EB30B380
         Drive Type: Data
         LD Acceleration Method: Con
```

A quick look at `fdisk` shows the drive ready for use:

```
root@lab-compute01:~# fdisk -l /dev/sdd
Disk /dev/sdd: 1.8 TiB, 2000365379584 bytes, 3906963632 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
Disklabel type: dos
Disk identifier: 0x44c89b3a

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdd1        2048 3907020799 3907018752  1.8T 83 Linux
```

If you have some thoughts or comments on this process, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.