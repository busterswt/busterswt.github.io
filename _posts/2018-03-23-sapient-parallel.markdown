---
title: "This Old Lisa: Sapient Technologies Dual-Port Parallel Card"
layout: post
date: 2018-03-23
image: /assets/images/2018-03-23-sapient-parallel/lisalogo.png
headerImage: true
tag:
- blog
- apple
- lisa
- vintage
- motherboard
blog: true
author: jamesdenton
description: Hands-on with the Sapient Technologies dual-port parallel card for the Apple Lisa

---

I recently acquired a second Apple Lisa 2/5 system in non-working condition, and part of its rehabilitation will involve replacing many of its internal components. Back in February, I installed and [reviewed](http://www.jimmdenton.com/sapient-motherboard/) a replacement motherboard for the Apple Lisa 1 and 2/5 personal computers. Luckily for me, Todd Meyer and the crew at [Sapient Technologies](https://www.facebook.com/SapientTechnologies/) have been hard at work and recently released a dual-port parallel card for the Apple Lisa and Macintosh XL that supports all known Lisa peripherals that utilize a parallel interface, including the Apple Dot Matrix printer and Apple ProFile hard disks as well as the [X/ProFile](http://vintagemicros.com/catalog/lisa-xprofile-hard-drive-emulator-p-282.html) from Sigma Seven Systems and VintageMicros. 

<!--more-->
The Apple Lisa and Macintosh XL series of machines rely on a parallel interface to provide connectivity to their respective storage devices. In the case of the Apple Lisa 1 and 2/5, an external parallel port is used to connect an Apple ProFile hard disk to the machine. On the Lisa 2/10 and Macintosh XL, the parallel interface exists as a header off the motherboard and connects to the internal Widget hard drive. Needless to say, additional parallel ports are a *must* for additional storage capabilities and other peripherals. 

# The parallel card
Apple released an expansion card for the Lisa known as the ***Parallel Interface*** that provided two external parallel ports and supported connectivity to Apple ProFile hard drives and other devices. 

<div class="side-by-side">
    <div class="toleft">
        <p>At the time, those peripherals included the Apple Dot Matrix printer and not much else. The Apple Lisa has three expansion slots, which means it can support up to three dual-port parallel cards connecting up to six additional hard disks to the system. Support for this configuration varies by operating system. At an original retail price of $3,499, six 5MB ProFile hard drives providing up to 30MB of additional storage would have cost a whopping $21,000!
</p>
    </div>
    
    <div class="toright">
        <center><img class="image" src="/assets/images/2018-03-23-sapient-parallel/original.png" alt="Cards installed"></center>
        <figcaption class="caption">These pop up on eBay from time to time in unknown working condition.</figcaption>
    </div>
    <br>
</div>
<br>
<div class="side-by-side">
    <div class="toleft">
    <center><img class="image" src="/assets/images/2018-03-23-sapient-parallel/sapient_400.png" alt="serial"></center>
        <figcaption class="caption">The Sapient Technologies dual-port parallel card carries the characteristic purple PCB that is the calling card of the brand.</figcaption>
      <br>
    </div>

    <div class="toright">
        <p>The Apple ProFile external hard disks came in two variants: a 5MB model and later, a 10MB model. Thanks to the MacWorks operating environment and other enhancements for the Lisa, users can take advantage of a wide array of software that is compatible with the Macintosh Plus. A single 5MB or even 10MB hard drive will likely not meet the storage needs of most users, resulting in the need for an expansion card that can provide additional storage capabilities. The Sapient dual-port parallel card fits the bill nicely, providing 100% compatibility with the original Apple card while using a combination of new and NOS (new old stock) components for added piece of mind.</p>
    </div>
    <br>
</div>
<br>
Like the original Apple dual-port parallel card, the Sapient card provides two DB-25 parallel interfaces for connecting devices to the system:

<figure>
  <img src="/assets/images/2018-03-23-sapient-parallel/orig_vs_sapient2_800.png" alt="side by side"/>
</figure>
  <figcaption>An *unmasked* original Apple parallel port card compared to a Sapient Technologies parallel card. </figcaption>
<br>
John Woodall at VintageMicros hand-builds every parallel card that goes out the door. The build quality is second to none, and each card is warrantied. 

# Installation

The installation process is very straightforward and can be accomplished with the following steps:

- Remove the rear panel of the Lisa
- Locate a free slot
- Slide out the silver spreader pin and turn the pin clockwise to spread the ZIF (zero insertion force) socket
- Insert the card until it hits the bumpstop
- Turn the pin counter-clockwise to close the socket and push the spreader pin back into the socket
- Reinstall the rear panel

The parallel card is ready for use out of the box and does not require drivers. At boot, any connected drive is eligible for use as a boot drive provided it has an operating system installed. Certain operating systems may be picky about which interface can be used (Xenix, for example) but the card is 100% compatible with any operating system that supports the original Apple card.

# Testing

When I first acquired this card I set out to verify that it supported a dual ProFile setup like the original Apple card it replaced. I wasn't surprised to find that it worked as expected with no errors or hiccups encountered. The card worked great with two 5MB Apple ProFile hard disks in both MacWorks and the Lisa Office System, and didn't skip a beat when I formatted one for use as the ```/usr``` partition for a Microsoft Xenix 3.0 setup.  

<figure>
  <img src="/assets/images/2018-03-23-sapient-parallel/macworks.png" alt="macworks"/>
</figure>
<br>
  <figcaption>Two Apple ProFile drives mounted in Macworks running System 6</figcaption>
  
<figure>
  <img src="/assets/images/2018-03-23-sapient-parallel/los.png" alt="side by side"/>
</figure>
<br>
  <figcaption>Two Apple ProFile drives mounted in the Lisa Office System version 3.0. The floppy cable is connected to a FloppyEmu (not pictured).</figcaption>

# Summary
Like many other components for the Apple Lisa, original parallel interface cards can fetch high prices thanks to their *vintageness* rather than their actual operating status. While not as prone to failure as a motherboard or I/O card might be, buying an original parallel interface from eBay or other unknown source can be an expensive gamble. The dual-port parallel interface card from Sapient Technologies is great alternative to the original while providing the same functionality, expansion capabilities, and compatibility. 

The LisaFAQ at [http://lisafaq.sunder.net/](http://lisafaq.sunder.net/) is a great resource for Lisa-related information, and all tips and tricks related to the Apple Parallel card apply to the Sapient card as well. For more info, check out the "[What are the characteristics of the Dual Parallel Port Card](http://lisafaq.sunder.net/lisafaq-hw-exp-2xpar_card.html)" section of the FAQ.

For interested buyers, the Sapient Technologies Dual-Port Parallel Card can be found at [VintageMicros](http://vintagemicros.com/catalog/lisa-dual-port-parallel-card-from-sapient-technologies-p-298.html).