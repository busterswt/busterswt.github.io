---
title: "This Old Lisa: Hands-on With the Sapient Technologies Lisa 1 & 2/5 Motherboard"
layout: post
date: 2018-02-26
image: /assets/images/2018-02-02-sapient-motherboard/lisalogo.png
headerImage: true
tag:
- blog
- apple
- lisa
- vintage
- motherboard
category: blog
blog: true
author: jamesdenton
description: Hands-on with the Sapient Technologies motherboard for the Apple Lisa

---

If you've been following the LisaList Google Group or other Apple Lisa news outlets lately, you may have seen mention of a group known as [Sapient Technologies](https://www.facebook.com/SapientTechnologies/) producing new hardware for the Apple Lisa. Sapient Technologies has some very interesting products in the pipeline, including:

- A Lisa 1 and Lisa 2/5 motherboard (referred to as Lisa 1/5 motherboard)
- A Lisa 2/10 and Macintosh XL motherboard
- A Lisa 1 and Lisa 2/5 Input/Output (I/O) board
- A Lisa 2/10 and Macintosh XL Input/Output (I/O) board
- A Lisa 1, 2/5, 2/10, Macintosh XL CPU board
- A 2-port parallel expansion card
- A 4-port serial expansion card for use with Microsoft Xenix

<!--more-->
To design and build these products, Todd Meyer of **Sapient Technologies** assembled a team consisting of himself and other Apple Lisa enthusiasts and professionals, including:

- James MacPhail of Sigma Seven Systems
- John Woodall of VintageMicros
- Rick Ragnini

Their purpose is simple: to extend the life of the Apple Lisa platform. 

These efforts have taken years to develop, and the resulting products could not have come at a better time. Thirty-five years after the introduction of the Apple Lisa and its offspring, the Apple Lisa 2 and the Macintosh XL, many of these machines have developed a knack for not functioning. Whether it's due to corrosion caused by leaking batteries or aging electrolytics, it's no secret that an Apple Lisa will need some ongoing TLC to remain in the stable.

My Lisa 2/5 has a few upgrades, including:

- Xlerator 18 from Sigma Seven Systems
- X/ProFile from Sigma Seven Systems
- Sun Remarketing SCSI Card
- Apple Dual-Port Parallel Card
- MacWorks Plus II PFG from Sigma Seven Systems

All of these combined makes for a computing experience not unlike a Macintosh Plus on steroids, and the perfect platform for testing Sapient's replacement components.

# The motherboard
The motherboard in the Apple Lisa is the backplane of the device. All of the goodness that makes a Lisa a *Lisa* connects to the motherboard. It's also located in a vulnerable position. Among their many achievements, Apple Lisa 1 and 2/5 machines are notorious for leaking batteries. The NiCD battery in an Apple Lisa was used to power parameter RAM when the machine was unplugged. Over the years, the contents of these batteries outgrew their shell and the corrosive material found its way onto the I/O board and the motherboard, destroying components and traces in its path. Folks have tried many methods in attempts to repair the damage, most involving vinegar baths, toothbrushes, and retracing, only to find the damage can't be stopped in the long run:

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/oldandbusted.png" alt="Old and Busted"/>
  <figcaption>A motherboard that's seen better days. Picture courtesy of classic-computers.org.nz.</figcaption>
</figure>

The Sapient Technologies motherboard for the Apple Lisa 1 and 2/5 is a drop-in replacement for the factory motherboard in both machines:

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/sapientboard.png" alt="Sapient Motherboard"/>
  <figcaption>A Sapient Technologies Lisa 1 and 2/5 replacement motherboard. Picture courtesy of Sapient Technologies.</figcaption>
</figure>

The Apple Lisa 2/10 and Macintosh XL are not prone to the same type of damage, simply because the batteries were removed from the design of their I/O boards. As a result, you're more likely to find working 2/10s and XLs in the wild.

## Rear connectors

As you'd expect, the motherboard includes two serial ports, a single parallel port, and a single mouse port, all in factory locations. The mouse port does not have a locking mechanism like the original motherboard, but is fully compatible with Lisa 1 and Lisa 2 mice as well as 9-pin Macintosh mice of the same era. I've successfully tested both a Lisa 1 mouse (A9M0050) and a Macintosh 512k mouse (M0100).

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/rearport.png" alt="Rear Ports"/>
  <figcaption>A look at the rear ports of the Sapient Technologies motherboard for the Apple Lisa 1 and 2/5.</figcaption>
</figure>

In a nod to the Lisa 2/10(XL) motherboard design, Sapient Technologies relocated the seldom-used video-out port and in its stead placed an 'Interrupt' button used to invoke the debugger. The reset button remains in the factory location. 

## Main slots

The replacement motherboard includes a slot for the CPU board and the Input/Output (I/O) board:

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/mainslots.png" alt="Main Slots"/>
  <figcaption>A look at the main slots of the Sapient Technologies motherboard for the Apple Lisa 1 and 2/5.</figcaption>
</figure>

Just like the original, the Sapient motherboard includes two RAM slots known as MEM1 and MEM2, and is completely compatible with factory Apple 512k memory boards as well as aftermarket RAM upgrades. I've successfully tested the following configurations:

- Single Apple 512k memory board (512k)
- Dual Apple 512k memory boards (1M)
- Single Sun Remarketing memory board (2M)

## Expansion slots

Like the factory motherboard, the Sapient motherboard includes 3x expansion port slots that are fully compatible with the few Lisa expansion cards on the market, past or present:

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/expansionslots.png" alt="Expansion Slots"/>
  <figcaption>A look at the expansion slots of the Sapient Technologies motherboard for the Apple Lisa 1 and 2/5.</figcaption>
</figure>

I've successfully tested the following expansion cards:

- Apple Dual-Port Parallel Card
- Sun Remarketing SCSI Card

## Drive Cage Fan

Like Macintosh systems of the era, the Lisa relies on convection cooling to keep things cool inside the chassis. With upgrades installed, the temperatures inside the chassis can rise to the point where random system crashes can occur on a regular basis. These crashes may manifest themselves as Address Errors, undocumented error codes, scrambled error messages, or even lockups. Even without upgrades, over time components exceed tolerences and can cause unexpected behavior. The Sapient motherboard includes a built-in fan to keep the temperatures down, which is a great thing for upgraded systems and even beneficial for stock applications:

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/cagefan.png" alt="Main Slots"/>
  <figcaption>A cage fan is included to keep the inside of the chassis cool and prolong the life of the components.</figcaption>
</figure>

The fan speed is not controlled by software, so it will run at peak speed. It's not really noticable, though, and the benefits outweight the additional noise. Besides, if you have a ProFile connected you'll never hear it! For those that don't want or need a fan, it can be disabled or removed.  

# Installation

The installation process involves removing the card cage from the machine, removing each card from the cage, and separating the cage frame from the motherboard. There are approximately twenty screws involved, give or take a couple. Once removed, the factory motherboard can be lifted out and placed in an anti-static bag for safe keeping.

<div class="side-by-side">
    <div class="toleft">
        <p>To install, simply place the Sapient motherboard onto the plate and replace the screws. This process is smoother than the removal of the factory motherboard, as the screws near the rear ports are no longer inhibted by factory nuts and the holes should line up perfectly. Replace the card cage frame and insert the cards in their proper location. Insert the card cage into the machine and secure the back plate.</p>
    </div>
    
    <div class="toright">
        <center><img class="image" src="/assets/images/2018-02-02-sapient-motherboard/cardsinstalled.png" alt="Cards installed"></center>
        <figcaption class="caption">The Xlerator 18 and Sun Remarketing RAM board installed.</figcaption>
    </div>
</div>

# Testing

Since acquiring this motherboard, I've put no less than 30 hours of actual usage on it, not including the more than 100 hours of idle power on-time. Equipped with my [X/ProFile](http://vintagemicros.com/catalog/product_info.php/products_id/282) (reviewed [here](http://www.jimmdenton.com/lisa-xprofile/)) and my [FloppyEmu](https://www.bigmessowires.com/floppy-emu/), I set off to test this motherboard with some real-world usage.

## Lisa Office System

<div class="side-by-side">
    <div class="toleft">
    <center><img class="image" src="/assets/images/2018-02-02-sapient-motherboard/lisadesktop.png" alt="Lisa OS"></center>
        <figcaption class="caption">The Lisa Office System Desktop</figcaption>
    </div>

    <div class="toright">
        <p>The Sapient Technologies Lisa 1/5 motherboard is electrically compatible with the factory Apple motherboard, so it was no surprise that it functioned as-expected with the Lisa Office System. In my tests, I successfully booted to LOS 2.0 and LOS 3.1, and even spent some time using LisaTerminal to communicate with my iMac over a serial cable (but failed to snap a picture!)</p>
    </div>
</div>

## MacWorks

<div class="side-by-side">
    <div class="toleft">
        <p>Macintosh operating systems can be used with MacWorks software for the Apple Lisa, which provides an environment from which early System software through System 7.5.5 can be used (with appropriate hardware). My Lisa 2/5 is equipped with a PFG for MacWorks Plus II, which allows up to System 7.5.5. For this testing, though, I stuck with System 7.1</p>
    </div>
    
    <div class="toright">
        <center><img class="image" src="/assets/images/2018-02-02-sapient-motherboard/system7-2.png" alt="System 7"></center>
        <figcaption class="caption">System 7.1 running on the Apple Lisa</figcaption>
    </div>
</div>

### Testing the serial ports

<div class="side-by-side">
    <div class="toleft">
    <center><img class="image" src="/assets/images/2018-02-02-sapient-motherboard/zterm.png" alt="serial"></center>
        <figcaption class="caption">Point-to-point connections. When you need it slowly and securely.</figcaption>
    </div>

    <div class="toright">
        <p>Using a USB to null serial adapter, I successfully transferred a few megabytes worth of binhex'd images from a 2015 iMac to the Lisa. Not exactly a fast process, but useful for those without mass storage devices.</p>
    </div>
</div>

### Testing general operations

As part of my testing procedure, many disk images were read from and written to the FloppyEmu without issue. The stock floppy drive had no issues, either. I went thru no less than five operating system installations, including Lisa Office System and Macintosh System 6.05, 6.08, 7.0, and 7.1.

<div class="side-by-side">
    <div class="toleft">
        <p>For grins, I fired up Adobe Illustrator. As expected, it ran without a hitch. The replacement motherboard won't make software work that didn't already work with the stock motherboard, but it <i>can</i> offer stability to certain operations affected by damaged components, such as printing or communications over the serial ports or data operations over the parallel interface.</p>
    </div>
    
    <div class="toright">
        <center><img class="image" src="/assets/images/2018-02-02-sapient-motherboard/illustrator.png" alt="Illustrator"></center>
        <figcaption class="caption">Illustrator may be overkill for text, but it's still cool.</figcaption>
    </div>
</div>

# Summary

It's no secret that developing new hardware for obsolete computing platforms is a labor of love and unlikely to make someone rich, which makes Sapient Technology's focus on extending the life of the Apple Lisa with quality products an admirable one. The Apple Lisa 1/5 motherboard from Sapient Technologies is a must-have item for those whose motherboards are beyond repair or whose failure is inevitable. Its compatibility with existing CPU, I/O, RAM, and expansion cards cannot be overstated. It *just works*. John Woodall at [VintageMicros](http://www.vintagemicros.com) hand-assembles these boards, and the quality is on-par with what I've come to expect from his work. Every port, slot, resistor, capacitor, or component has a purpose. Every solder joint is clean. Every trace is intact. If you're restoring an Apple Lisa 1 or a Lisa 2/5, and you're looking for a motherboard that you ***know*** will work, you can't go wrong with the Sapient Technologies Lisa 1/5 Motherboard.

For interested buyers, the Sapient Technologies Lisa 1/5 Motherboard can be found at [VintageMicros](http://vintagemicros.com/catalog/product_info.php/products_id/291). 

<figure>
  <img src="/assets/images/2018-02-02-sapient-motherboard/lisalogo2.png" alt="Lisa!"/>
  <figcaption>Picture courtesy of futurezone.de.</figcaption>
</figure>

