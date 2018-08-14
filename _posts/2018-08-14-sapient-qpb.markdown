---
title: "Sapient Technologies Quad-Port Serial Expansion Card for the Apple Lisa"
layout: post
image: /assets/images/2018-03-23-sapient-parallel/lisalogo.png
headerImage: true
tag:
- blog
- apple
- lisa
- vintage
- motherboard
category: blog
blog: true
published: true
author: jamesdenton
description: Hands-on with the Sapient Technologies quad-port serial expansion card for the Apple Lisa

---

My foray into the Apple Lisa has led me to people and communities looking to breath new life into the vintage system. This is nothing new, of course. The platform saw CPU upgrades introduced years after it was discontinued, and folks brought out hard disk replacements in the early 2000's to replace aging Apple ProFile drives. The Lisa holds a special place in Apple lore and in the hearts of enthusiasts and collectors around the world.
<!--more-->

So, when I found out that a quad-port serial expansion card was in the works, I had to know more. John Woodall from VintageMicros explained that the new quad-port board, or QPB, was a modernized version of a serial card for the Apple Lisa that was manufactured by Tecmar, a computer peripheral company that created products for many different systems and platforms in the 1980s and early 1990s. Rather than providing additional serial interfaces to users of the Lisa's main operating system, the Lisa Office System, the card was to be used within the Microsoft/SCO XENIX operating system and later, UniSoft's UniPlus+ UNIX. 

![Hub](/assets/images/2018-08-14-sapient-qpb/hub.png)

At the time, UNIX had been around for nearly a decade, but was mostly used on large mainframes and minicomputers like the PDP/11. In the early 80's, as microcomputers became more powerful, you could find vendors porting UNIX and UNIX-based operating systems like XENIX to less-expensive systems like the Apple Lisa and other 68000 or 8086-based systems, among others.

Thanks to Sapient Technologies, this unique and somewhat unobtainium expansion card has been made available to the Lisa community. But how would one use it? 

# The card
The Quad-Port Serial Expansion Board, or QPB, offers a total of four serial interfaces that can be used to connect the Lisa to different terminals or terminal emulators, as well as serial printers and modems, within a supported operating system. 

![Serial Card](/assets/images/2018-08-14-sapient-qpb/serialcard.png)

Two external RS232 DB25 serial interfaces can be connected to modern systems and peripherals using cables and adapters. The card also includes two internal headers that can be exposed using DB25-to-ribbon cable adapters that are provided with the card.

Using a USB-to-serial adapter, users can connect modern systems to the Lisa as terminal emulators, which allows them to login and interface with the operating system running on the Lisa as if they were seated at the console:

![Z-Term MacOS](/assets/images/2018-08-14-sapient-qpb/ztermmac.png)

Or, users can adopt a fully-retro setup and connect using vintage systems, like the Macintosh 128k, using adapters provided by VintageMicros:

![MacTerminal](/assets/images/2018-08-14-sapient-qpb/macterminal.png)

# Installation

The Quad-Port Serial Expansion Board comes with a guide that walks the reader through the process of installing the card in an Apple Lisa 2. Like most expansion cards, the installation process is very straightforward and can be accomplished with the following steps:

- Remove the rear panel of the Lisa
- Locate slot number 3 (closest to the I/O board)
- Slide out the silver spreader pin and turn the pin clockwise to spread the ZIF (zero insertion force) socket
- Insert the card until it hits the bumpstop
- Turn the pin counter-clockwise to close the socket and push the spreader pin back into the socket
- Reinstall the rear panel

![Case Pic](/assets/images/2018-08-14-sapient-qpb/serialcase.jpg)

In addition to installing the serial card, a compatible operating system must be installed for the card to be functional. VintageMicros provides a kit that includes Microsoft/SCO XENIX and UniSoft UniPlus+ UNIX on compact flash cards that are compatible with the [X/ProFile](http://vintagemicros.com/catalog/lisa-xprofile-hard-drive-emulator-p-282.html) hard disk system. Instructions for installing the compact flash card(s) is also provided. Users are also free to install the operating systems themselves on original Apple ProFile or Widget drives using original guides available on the Internet.


# UNIX on the Lisa

Many modern operating systems are based on UNIX, and macOS and iOS are no exceptions. Anyone familiar with using the Terminal application on a Mac, and especially those fluent in Linux, should feel right at home within Microsoft XENIX or UniPlus+ UNIX. That said, there are some notable differences between older UNIXes and today's operating systems. 

XENIX for the Lisa is based on AT&T's UNIX System III released in 1982, while UniPlus+ UNIX is a little newer and based on AT&T UNIX System V first released in 1983. Neither include a TCP/IP network stack - the workhorse of the modern Internet - though it was available at the time on certain UNIX distributions. 

In their day, machines were connected to networks using modems and to some extent direct serial connections. UNIX offered the UUCP suite of tools to provide file and mail operations between hosts. Users could copy files between machines and exchange mail messages with users on other systems. In fact, MacOS releases as recent as High Sierra include UUCP software that can be configured to communicate with the Lisa running UNIX or XENIX, allowing the Lisa to send files and even electronic mail! Connections to bulletin-board systems were likely possible as well, though none of the BBSes are around to allow one to recreate that magic. 

Microsoft offered a spreadsheet and word processor for XENIX systems, and both were available for the Apple Lisa. The spreadsheet, known as Multiplan, is included with the XENIX distribution provided by VintageMicros:

![Multiplan](/assets/images/2018-08-14-sapient-qpb/multiplan.png)

The word processor, Microsoft Lyrix, is not included but can be found at [bitsavers.org](http://www.bitsavers.org).

The UniPlus+ UNIX distribution includes two UNIX games, Mastermind and Wump, that I'm sure provided countless hours of entertainment to users of the day:

![Wump](/assets/images/2018-08-14-sapient-qpb/wump.png)

Other original games from BSD UNIX can be found on the Internet, including [here](https://github.com/weiss/original-bsd), that can be directly compiled or ported using the included C compiler and associated tools.

# Summary

Using UNIX-based operating systems on the Apple Lisa is a real treat, and further demonstrates the Lisa's flexibility and backwards compatibility of modern UNIX-based operating systems. The Quad-Port Serial Expansion Board enables the Lisa to act as a communications hub for up to four directly-connected hosts, and users can interact many more using an intermediate device like a Raspberry Pi to route UUCP over the Internet.

The 100+ page color User's Guide includes information on basic administrative tasks, printing, and more, within the XENIX and UNIX operating systems, and even includes information on interfacing with terminal emulators running on new and vintage Macs and PCs running Windows. Instructions for configuring UUCP on High Sierra are included for those willing to walk on the wild side. 

For more information on the Sapient Technologies Quad-Port Serial Expansion board and the accompanying kit, contact John Woodall of [VintageMicros](http://vintagemicros.com/).