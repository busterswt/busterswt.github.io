---
title: "Configuring GPU Passthrough in OpenStack"
layout: post
date: 2019-05-09
image: /assets/images/2019-05-09-gpu-offloading-openstack/zoom-zoom.png
headerImage: true
tag:
- gpu
- passthrough
- openstack-ansible
- nvidia
- openstack
category: blog
blog: true
author: jamesdenton
description: "Configuring GPU Passthrough in OpenStack"
---

As part of my effort to "cut the cord", I recently install Plex Media Server onto a virtual machine instance running in my home OpenStack environment. The compute node is a few generations old, and while perfectly capable of running many different workloads, transcoding Ultra HD (4K) content is not one of them. 

To combat this, I recently installed an NVIDIA Quadro P2000 video card with the goal of passing it through to the virtual machine instance to improve the viewing experience.
<!--more-->

My OpenStack cluster consists of the following:

- 1x Infrastructure Node: HP DL360e G8 (2x 1.8 Ghz Intel Xeon 2450L)
- 1x Compute Node: HP DL380p G8 (2x 3.3 Ghz Intel Xeon 2667 v2)

The virtual machine instance running Plex Media Server has the following specs:

- 6 cores
- 8 GB RAM
- 40 GB Disk

All media is stored on a NAS and accessed over NFS.


# Preparation

In a recent post, I documented the process of procuring the proper riser and installing the GPU into this server. This process will vary from machine to machine, of course.

To pass a GPU through to virtual machines, you will want to enable VT-d extentions in the BIOS. This, too, will vary from platform to platform.

Next, update the grub configuration at `/etc/default/grub` to ensure IOMMU support is enabled. For Intel, boot the machine, and append `intel_iommu=on` to the end of the `GRUB_CMDLINE_LINUX` line in the grub configuration file:

```
$ sudo nano /etc/default/grub
...
GRUB_CMDLINE_LINUX="splash=quiet console=tty0 ... intel_iommu=on
...
```

For AMD, boot the machine, and append `amd_iommu=on` to the end of the `GRUB_CMDLINE_LINUX` line in the grub configuration file:

```
$ sudo nano /etc/default/grub
...
GRUB_CMDLINE_LINUX="splash=quiet console=tty0 ... amd_iommu=on
...
```

Refresh the grub configuration and reboot the host:

```
$ sudo update-grub
$ reboot
```

# Driver Shenanigans

Once the host has been rebooted, the next step in preparing the GPU for passthrough is to ensure the proper drivers are configured. A few useful pieces of information that will be needed as we progress through the process include the following:

- Vendor ID
- Product ID
- PCI Bus ID

These can all be determined with the `lspci` command shown here:

```
$ sudo lspci -nn | grep NVIDIA
24:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106GL [Quadro P2000] [10de:1c30] (rev a1)
24:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
```

In this example, the PCI Bus ID is `24:00.0`, the Vendor ID is `10de` and the Product ID is `1c30`. 

There is a good chance that the kernel has detected the GPU and loaded a video card driver. In the case of NVIDIA cards, this likely means the open source `nouveau` driver, as shown here:

```
$ sudo lspci -s 24:00.0 -k
24:00.0 VGA compatible controller: NVIDIA Corporation GP106GL [Quadro P2000] (rev a1)
	Subsystem: Dell GP106GL [Quadro P2000]
	Kernel driver in use: nouveau
	Kernel modules: nvidiafb, nouveau
```

While the video card is bound to a graphics driver, GPU passthrough will not be possible.

## Blacklist

In order to ensure the graphics drivers are not loaded, we need to blacklist them. Create and edit a dedicated blacklist file at `/etc/modprobe.d/blacklist-nvidia.conf` and add the following contents:

```
blacklist nouveau
blacklist nvidiafb
```

## Whitelist

Next, we need to ensure that the `vfio-pci` driver gets loaded and binds the GPU. Create and edit a dedicated file at `/etc/modprobe.d/vfio.conf` and add the following contents:

```
options vfio-pci ids=10de:1c30,10de:10f1
```

Then, add the `vfio-pci` driver to the `/etc/modules-load.d/modules.conf` file to ensure it's loaded:

```
8021q
br_netfilter
dm_multipath
dm_snapshot
ebtables
ip6table_filter
ip6_tables
ip_tables
...
vfio-pci
```

Reboot the host.

```
$ sudo reboot
```

Once the host has returned, you can verify the proper driver has been bound using the following `lspci` command:

```
$ sudo lspci -s 24:00.0 -nnk
24:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106GL [Quadro P2000] [10de:1c30] (rev a1)
	Subsystem: Dell GP106GL [Quadro P2000] [1028:11b3]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau
```

The `Kernel driver in use: vfio-pci` line indicated the proper driver is now in place.

# Nova changes

The OpenStack Compute service needs to be configured in two place in order to recognize and utilize the GPU.

First, configure a PCI passthrough whitelist on the compute node where the GPU resides. Update the `[PCI]` section of the `/etc/nova/nova.conf` file with the following:

```
[PCI]
passthrough_whitelist: { "vendor_id": "10de", "product_id": "1c30" }
```

Ensure that the `vendor_id` and `product_id` match that retrieved earlier in the process. The whitelist can be limited to referencing a particular PCI slot if needed, but since I only have a single card installed, that isn't really necessary. Restart the `nova-compute` service:

```
$ sudo systemctl restart nova-compute
```

Next, configure a PCI alias on the node hosting the Nova API service. Update the `[PCI]` section of the `/etc/nova/nova.conf` file with the following:

```
[PCI]
alias: { "vendor_id":"10de", "product_id":"1c30", "device_type":"type-PCI", "name":"quadro-p2000" }
```

Restart the Nova API service:

```
$ sudo systemctl restart nova-api
```

Lastly, ensure that the Nova scheduler has been configured with a PCI Passthrough filter. You can verify `PciPassthroughFilter` is listed in `enabled_filters` and/or `available_filters` in the `/etc/nova/nova.conf` file, as shown here, and add it if necessary:

```
[filter_scheduler]
enabled_filters = RetryFilter, AvailabilityZoneFilter, ComputeFilter, ComputeCapabilitiesFilter, ImagePropertiesFilter, ServerGroupAntiAffinityFilter, ServerGroupAffinityFilter, PciPassthroughFilter
available_filters = nova.scheduler.filters.all_filters
```

Restart the Nova scheduler service if necessary:

```
$ sudo systemctl restart nova-scheduler
```

# Create a flavor

Nova uses flavor metadata and properties to determine what resources should be associated with an instance. In the case of GPU passthrough, one must configure a flavor with the `pci_passthrough:alias` property. In the following example, I've created a new flavor with 6 vCPUs, 8 GB of RAM, a 40 GB disk, and a `pci_passthrough:alias` property referencing the alias we configured earlier:

```
openstack flavor create \
--vcpus 6 \
--ram 8192 \
--disk 40 \
--property "pci_passthrough:alias"="quadro-p2000:1" \
6-8-40-gpu
```

In `quadro-p2000:1`, the `quadro-p2000` directly references the alias name, while the `1` instructs Nova that a single GPU should be assigned. 

# Boot an instance

To test the new flavor and new GPU passthrough capabilities, spin up a virtual machine instance using the new flavor, as shown here:

```
openstack server create \
--flavor 6-8-40-gpu \
--image ubuntu-bionic \
--network LAN \
--key-name imac-rsa \
--security-group plex \
plex-pgu-server
```

After a brief moment, the server should be `ACTIVE`:

```
root@lab-infra01-utility-container-cee33e10:~# openstack server list
+--------------------------------------+-------------------+---------+-------------------+---------------+---------------+
| ID                                   | Name              | Status  | Networks          | Image         | Flavor        |
+--------------------------------------+-------------------+---------+-------------------+---------------+---------------+
| 854ba22b-52c0-44a1-ab66-b78eba142bc5 | plex-pgu-server   | ACTIVE  | LAN=192.168.2.111 | ubuntu-bionic | 6-8-40-gpu    |
+--------------------------------------+-------------------+---------+-------------------+---------------+---------------+
```

Login to the machine and verify the GPU is recognized:

```
ubuntu@plex-pgu-server:~$ lspci | grep NVIDIA
00:06.0 VGA compatible controller: NVIDIA Corporation GP106GL [Quadro P2000] (rev a1)
```

For best performance, updated drivers can be installed from the upstream Ubuntu repository or from NVIDIA's website. In this case, I've chosen to install drivers from the upstream repo using the `ubuntu-drivers` utility:

```
ubuntu@plex-pgu-server:~$ sudo apt update
ubuntu@plex-pgu-server:~$ sudo apt install ubuntu-drivers-common

ubuntu@plex-pgu-server:~$ ubuntu-drivers devices
== /sys/devices/pci0000:00/0000:00:06.0 ==
modalias : pci:v000010DEd00001C30sv00001028sd000011B3bc03sc00i00
vendor   : NVIDIA Corporation
model    : GP106GL [Quadro P2000]
driver   : nvidia-driver-390 - distro non-free recommended
driver   : xserver-xorg-video-nouveau - distro free builtin
```

The utility recommends the `nvidia-driver-390` driver, and even though there may be newer drivers available, this works for my purposes. 

```
ubuntu@plex-pgu-server:~$ sudo apt install nvidia-driver-390
```

Once the drivers have been install, reboot the instance:

```
ubuntu@plex-pgu-server:~$ sudo reboot
```

After rebooting the instance, verify the new drivers have been installed using `lspci` and `nvidia-smi`:

```
ubuntu@plex-pgu-server:~$ sudo lspci -nnk -s 00:06.0
00:06.0 VGA compatible controller [0300]: NVIDIA Corporation GP106GL [Quadro P2000] [10de:1c30] (rev a1)
	Subsystem: Dell GP106GL [Quadro P2000] [1028:11b3]
	Kernel driver in use: nvidia <---
	Kernel modules: nvidiafb, nvidia_drm, nvidia

ubuntu@plex-pgu-server:~$ nvidia-smi
Wed May  8 18:08:14 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.116                Driver Version: 390.116                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Quadro P2000        Off  | 00000000:00:06.0 Off |                  N/A |
|  0%   38C    P0    19W /  75W |      0MiB /  5059MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Voila! Now it's off to the races!

# Benchmarking

The best way to get a *BEFORE* and *AFTER* is to kick off the transcoder. For this, I fired up the AppleTV 4 (HD) to force Plex to transcode an 4K movie to 1080p. To do this, you'd need a Plex Pass to enable Hardware Acceleration, but there may be alternative methods of verifying.

Here's a look at the CPU statistics before playing the movie:

![transcode cpu noop](/assets/images/2019-05-09-gpu-offloading-openstack/transcode-noop.png)

Once the movie started playing, CPU usage shot up over 85%:

![transcode cpu software](/assets/images/2019-05-09-gpu-offloading-openstack/transcode-software.png)

A look at the Plex dashboard reflected software-based transcoding (note the lack of 'hw'):

![dashboard software](/assets/images/2019-05-09-gpu-offloading-openstack/status-software.png)

We can see the Quadro P2000 just sitting there, unutilized:

```
Wed May  8 19:04:40 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.116                Driver Version: 390.116                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Quadro P2000        Off  | 00000000:00:06.0 Off |                  N/A |
|  0%   38C    P0    19W /  75W |      0MiB /  5059MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Enabling Hardware Acceleration in the Plex server settings and restarting the movie reflected an significant drop in CPU utilization:

![transcode cpu hardware](/assets/images/2019-05-09-gpu-offloading-openstack/transcode-hardware.png)

A drop from 80% to ~50%? Not shabby, but not great. A look at the Plex dashboard reflected hardware-based transcoding was in effect:

![dashboard hardware](/assets/images/2019-05-09-gpu-offloading-openstack/status-hardware.png)

We can see the Quadro P2000 barely trying:

```
Wed May  8 19:11:03 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.116                Driver Version: 390.116                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Quadro P2000        Off  | 00000000:00:06.0 Off |                  N/A |
| 49%   42C    P0    19W /  75W |    161MiB /  5059MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0     26847      C   /usr/lib/plexmediaserver/Plex Transcoder     151MiB |
+-----------------------------------------------------------------------------+
```

Turns out, fully-offloaded transcoding isn't yet supported in Plex Media Server for Linux. Further research led me to some FFmpeg flags that would really make this card shine.  To implement these, however, one must a) wait for a future update to Plex Media Server or b) create a wrapper script to implement the flags. I chose the latter.

## The script

To offload both encoding *and* decoding, a wrapper script should be implemented that adds an addition flag to enable the NVDEC engine.

First, rename the `Plex Transcoder` to `Plex Transcoder Orig`:

```
ubuntu@plex-pgu-server:~$ cd /usr/lib/plexmediaserver
ubuntu@plex-pgu-server:~$ sudo mv Plex\ Transcoder Plex\ Transcoder\ Orig
```

Then, create a file named `Plex Transcoder` with the following contents:

```
#!/bin/bashmarap=$(cut -c 10-14 <<<"$@")if [ $marap == "mpeg4" ]; then     exec /usr/lib/plexmediaserver/Plex\ Transcoder2 "$@"else     exec /usr/lib/plexmediaserver/Plex\ Transcoder\ Orig -hwaccel nvdec "$@"fi
```

Next, set the file to be executable:

```
ubuntu@plex-pgu-server:~$ sudo chmod 755 /usr/lib/plexmediaserver/Plex\ Transcoder
```

The changes should be noticed immediately. Restarting the stream, we can see CPU utilization drop to below 20%:

![transcode cpu wrapper](/assets/images/2019-05-09-gpu-offloading-openstack/transcode-wrapper.png)

Now we're cooking with gas! A quick look at `nvidia-smi` shows the GPU barely breaking a sweat:

```
Wed May  8 19:25:14 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.116                Driver Version: 390.116                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Quadro P2000        Off  | 00000000:00:06.0 Off |                  N/A |
| 49%   41C    P0    21W /  75W |   1034MiB /  5059MiB |     11%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0     27118      C   ...ib/plexmediaserver/Plex Transcoder Orig  1024MiB |
+-----------------------------------------------------------------------------+
```

# Summary

If you have 4K content, chances are your viewers would benefit from **Direct Play** functionality only possible with true 4K-capable devices. Transcoding 4K to something like 1080p will likely result in some color manipulation that astute viewers would dislike, but for kids, it's good enough. Letting the Xeons try to transcode the media results in a lousy viewing experience, so the GPU is worth it to me. I also consider reducing the amount of vCPUs reserved for Plex in my lab as a win. 

If you have some thoughts or comments on this process, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton.