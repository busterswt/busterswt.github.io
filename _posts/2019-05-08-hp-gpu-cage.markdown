---
title: "Installing a GPU in an HP ProLiant DL380p G8"
layout: post
date: 2019-05-08
image: /assets/images/2019-05-08-hp-gpu-cage/radeon_fire.jpg
headerImage: true
tag:
- gpu
- hp
- proliant
- dl380p
- nvidia
category: blog
blog: true
author: jamesdenton
description: "Installing a GPU in an HP ProLiant DL380p G8"
---

Over the last few weeks, I've been slowly implementing alternative media sources in the house since "cutting the cord" and cancelling DirecTV service. Clients include an AppleTV 4 (4K) in the living room and an AppleTV 4 (HD) in the master bedroom, along with an old Amazon FireTV in the basement and some cell phones and tablets. Quite a variety of *things*.

We've used subscription apps like Netflex and YouTube Red/Premium for a long time now, and I implemented a Plex server that would allow me to convert existing BluRay and DVD media as well as leverage an HD Home Run w/ an HD antenna.
<!--more-->

If you've ever tried to play 4K media on a non-4K device, Plex really struggles to transcode that stream to something supported, like 1080p, if you don't have hardware transcoding enabled on a modern CPU or GPU. I'm running Plex Media Server on a VM installed in my home OpenStack cluster, and the Xeon hardware I'm using doesn't really have the grunt to transcode that media efficiently, especially in a virtual machine.

## Enter GPU Passthrough

The HP DL380p G8 can support multiple GPUs with the proper PCI riser. Originally, I had 2 risers installed - each with 3x slots as shown here:

![original cage](/assets/images/2019-05-08-hp-gpu-cage/original_cage.jpg)

Depending on the GPU, a PCIe 3.0 x16 slot may be required. In my risers, there is only one of these slots available and it sits along the top edge. This configuration prevents the use of 2-slot GPUs, which makes up the majority of 'consumer' cards like the Nvidia GTX 1050 Ti and others in its class. 

To resolve this, I turned to the HP QuickSpecs PDF and tried to find a suitable riser. *Not a simple task*. There are a few varieties of risers available for the DL380p G8, but for this project there's only one that really matters:

**662885-B21**

Also known as the **HP DL380P GEN8 DOUBLE WIDE RSR CAGE KIT** [[Amazon]](https://amzn.to/2VaeL3T)

![cage_withcover](/assets/images/2019-05-08-hp-gpu-cage/cage_withcover.jpg)

The 662885-B21, as it's affectionately known, is a PCI riser cage that contains 2x PCIe 3.0 x16 slots. There are not a lot of good and/or accurate pics available online, which meant I had to take a chance on this one. Luckily, it was the right one.

## The hardware

Removing the cage from the riser reveals two slots:

![cage_nocover](/assets/images/2019-05-08-hp-gpu-cage/cage_nocover.jpg)

It should be possible to install a double-width card, but I found some cards have fans that cause the card to exceed the width of two slots and hit the edge of the riser. Given some consumer cards have two fans and the riser has ventilation on one side, it's not a practical solution. As a result, I went with the single-width NVIDIA Quadro P2000 [[Amazon]](https://amzn.to/2W01Oyh):

![quadro_p2000](/assets/images/2019-05-08-hp-gpu-cage/quadro_p2000.jpg)

The card could fit in either slot and still have some ventilation, but I decicded to install it in the slot furthest away from the edge of the riser. This would leave room for a smaller card in-between, if needed:

![card_cage](/assets/images/2019-05-08-hp-gpu-cage/card_cage.jpg)

I installed the riser in the second PCI bay, which requires a 2nd CPU:

![riser_installed](/assets/images/2019-05-08-hp-gpu-cage/riser_installed.jpg)

## Powering up

My server sits in the basement, which means I rely on the remote console via iLo for OOB management. On power up, everything looked good:

![ilo_good](/assets/images/2019-05-08-hp-gpu-cage/ilo_goodr.png)

A few minutes later, though, and things were taking a turn:

![ilo_blank](/assets/images/2019-05-08-hp-gpu-cage/ilo_blank.png)

After the first splash screen disappeared, there was *nothing*. No feedback, no errors, and no way to modify the BIOS. After a few reboots with no change in behavior, it dawned on me that something else may have been going on. I ran a ping against the host to see if there was any life. Lo and behold...

```
Request timeout for icmp_seq 337
Request timeout for icmp_seq 338
Request timeout for icmp_seq 339
Request timeout for icmp_seq 340
Request timeout for icmp_seq 341
Request timeout for icmp_seq 342
64 bytes from 10.20.0.31: icmp_seq=343 ttl=64 time=2.704 ms
64 bytes from 10.20.0.31: icmp_seq=344 ttl=64 time=5.621 ms
64 bytes from 10.20.0.31: icmp_seq=345 ttl=64 time=3.555 ms
64 bytes from 10.20.0.31: icmp_seq=346 ttl=64 time=3.849 ms
64 bytes from 10.20.0.31: icmp_seq=347 ttl=64 time=5.726 ms

```

Unbeknownst to me, the host was transferring video (including iLo) to the GPU. So, the server was booting but no video was passed through to iLo. I'm sure if a monitor had been connected to the GPU I would've seen something. Silly, yet expected, behavior (supposedly). This would have been much trickier to diagnose had the server not been located onsite.

## Verification

Once I was able to login to the machine, I verified that the card was at least recognized by the host:

```
root@lab-compute01:~# lspci -vvv -nn | grep NVIDIA
24:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106GL [Quadro P2000] [10de:1c30] (rev a1) (prog-if 00 [VGA controller])
24:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)

```

From here, you can install drivers from the upstream Ubuntu repository or from [www.nvidia.com](https://www.nvidia.com). In a future writeup, I'll document how to implement PCI passthrough and pass the GPU to a virtual machine.
