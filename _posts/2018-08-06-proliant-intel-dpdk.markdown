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

When attempting to add the port, an error was logged in `dmesg` that seemed rather ominous:

```
[  708.044615] vfio-pci 0000:03:00.0: Device is ineligible for IOMMU domain attach due to platform RMRR requirement.  Contact your platform vendor.
```

Some searching turned up some excellent, albeit confusing, resources on the error:

- [https://access.redhat.com/sites/default/files/attachments/rmrr-wp1.pdf](https://access.redhat.com/sites/default/files/attachments/rmrr-wp1.pdf)
- [https://support.hpe.com/hpsc/doc/public/display?docId=emr_na-c04781229&sp4ts.oid=5249566](https://support.hpe.com/hpsc/doc/public/display?docId=emr_na-c04781229&sp4ts.oid=5249566)

The TL;DR here is that current Linux kernels (3.16+) no longer allow assignment of devices associated with VT-d Reserved Memory Reporting Regions (RMRRs).  These regions are used by some platform vendors, including HP, for side-band communication between I/O devices and the management controller. These regions are incompatible with PCI device assignment and stops our attempt to use the NICs for DPDK dead in its tracks. This does not seem to impact SR-IOV, however.

# So, how do we fix it?

Well, the HP document alludes to some utilities that can be used to modify the BIOS within the operating system, as such controls are not available with the ROM-based setup utility. These utilities can be used to exclude PCI slots from the RMRR mapping and should allow both HP and third-party Intel NICs to work with DPDK.

The following instructions can be used to install said utilities from an HP repository (last tested on Ubuntu 16.04 LTS):

```
cat <<EOF > /etc/apt/sources.list.d/stk.list
deb https://downloads.linux.hpe.com/SDR/repo/stk/ xenial/current non-free
EOF

curl http://downloads.linux.hpe.com/SDR/hpPublicKey1024.pub | apt-key add -
curl http://downloads.linux.hpe.com/SDR/hpPublicKey2048.pub | apt-key add -
curl http://downloads.linux.hpe.com/SDR/hpPublicKey2048_key1.pub | apt-key add -
curl http://downloads.linux.hpe.com/SDR/hpePublicKey2048_key1.pub | apt-key add -

apt update
apt install hp-scripting-tools
```

Once the utilities are installed, a special file should be downloaded from HP that contains the `conrep` configuration information for managing individual PCI slots:

```
wget -O conrep_rmrds.xml https://downloads.hpe.com/pub/softlib2/software1/pubsw-linux/p1472592088/v95853/conrep_rmrds.xml
```
Next, identify the **physical slot** that the NIC is plugged in to using the `lspci` command shown here:

```
lspci -vvv
```

Amongst the PCI data returned you will find the information pertaining to the NIC in question. On my machine, the NIC can be identified by bus address `03:00.x`:

```
03:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
	Subsystem: Intel Corporation Ethernet Server Adapter X520-2
	Physical Slot: 2
	Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr+ Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Interrupt: pin A routed to IRQ 16
	Region 0: Memory at f6380000 (64-bit, prefetchable) [disabled] [size=512K]
	Region 2: I/O ports at 6000 [disabled] [size=32]
	Region 4: Memory at f6370000 (64-bit, prefetchable) [disabled] [size=16K]
	Capabilities: [40] Power Management version 3
		Flags: PMEClk- DSI+ D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold-)
		Status: D3 NoSoftRst- PME-Enable- DSel=0 DScale=1 PME-
	Capabilities: [50] MSI: Enable- Count=1/1 Maskable+ 64bit+
		Address: 0000000000000000  Data: 0000
		Masking: 00000000  Pending: 00000000
	Capabilities: [70] MSI-X: Enable- Count=64 Masked-
		Vector table: BAR=4 offset=00000000
		PBA: BAR=4 offset=00002000
	Capabilities: [a0] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 512 bytes, PhantFunc 0, Latency L0s <512ns, L1 <64us
			ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset+
		DevCtl:	Report errors: Correctable- Non-Fatal- Fatal- Unsupported-
			RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+ FLReset-
			MaxPayload 128 bytes, MaxReadReq 4096 bytes
		DevSta:	CorrErr+ UncorrErr- FatalErr- UnsuppReq+ AuxPwr- TransPend-
		LnkCap:	Port #0, Speed 5GT/s, Width x8, ASPM L0s, Exit Latency L0s <1us, L1 <8us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp-
		LnkCtl:	ASPM Disabled; RCB 64 bytes Disabled- CommClk+
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed 5GT/s, Width x4, TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+, LTR-, OBFF Not Supported
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis-, LTR-, OBFF Disabled
		LnkCtl2: Target Link Speed: 5GT/s, EnterCompliance- SpeedDis-
			 Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
			 Compliance De-emphasis: -6dB
		LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete-, EqualizationPhase1-
			 EqualizationPhase2-, EqualizationPhase3-, LinkEqualizationRequest-
	Capabilities: [100 v1] Advanced Error Reporting
		UESta:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq+ ACSViol-
		UEMsk:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq+ ACSViol-
		UESvrt:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		CESta:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- NonFatalErr+
		CEMsk:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- NonFatalErr-
		AERCap:	First Error Pointer: 00, GenCap+ CGenEn- ChkCap+ ChkEn-
	Capabilities: [140 v1] Device Serial Number 00-1b-21-ff-ff-53-1e-70
	Capabilities: [150 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS-, Next Function: 1
		ARICtl:	MFVC- ACS-, Function Group: 0
	Capabilities: [160 v1] Single Root I/O Virtualization (SR-IOV)
		IOVCap:	Migration-, Interrupt Message Number: 000
		IOVCtl:	Enable- Migration- Interrupt- MSE- ARIHierarchy-
		IOVSta:	Migration-
		Initial VFs: 64, Total VFs: 64, Number of VFs: 0, Function Dependency Link: 00
		VF offset: 384, stride: 2, Device ID: 10ed
		Supported Page Size: 00000553, System Page Size: 00000001
		Region 0: Memory at 00000000f7f00000 (64-bit, non-prefetchable)
		Region 3: Memory at 00000000f7e00000 (64-bit, non-prefetchable)
		VF Migration: offset: 00000000, BIR: 0
	Kernel driver in use: vfio-pci
	Kernel modules: ixgbe

03:00.1 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
	Subsystem: Intel Corporation Ethernet Server Adapter X520-2
	Physical Slot: 2
	Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr+ Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Interrupt: pin B routed to IRQ 17
	Region 0: Memory at f6280000 (64-bit, prefetchable) [disabled] [size=512K]
	Region 2: I/O ports at 6020 [disabled] [size=32]
	Region 4: Memory at f6270000 (64-bit, prefetchable) [disabled] [size=16K]
	Capabilities: [40] Power Management version 3
		Flags: PMEClk- DSI+ D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold-)
		Status: D3 NoSoftRst- PME-Enable- DSel=0 DScale=1 PME-
	Capabilities: [50] MSI: Enable- Count=1/1 Maskable+ 64bit+
		Address: 0000000000000000  Data: 0000
		Masking: 00000000  Pending: 00000000
	Capabilities: [70] MSI-X: Enable- Count=64 Masked-
		Vector table: BAR=4 offset=00000000
		PBA: BAR=4 offset=00002000
	Capabilities: [a0] Express (v2) Endpoint, MSI 00
		DevCap:	MaxPayload 512 bytes, PhantFunc 0, Latency L0s <512ns, L1 <64us
			ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset+
		DevCtl:	Report errors: Correctable- Non-Fatal- Fatal- Unsupported-
			RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+ FLReset-
			MaxPayload 128 bytes, MaxReadReq 4096 bytes
		DevSta:	CorrErr+ UncorrErr- FatalErr- UnsuppReq+ AuxPwr- TransPend-
		LnkCap:	Port #0, Speed 5GT/s, Width x8, ASPM L0s, Exit Latency L0s <1us, L1 <8us
			ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp-
		LnkCtl:	ASPM Disabled; RCB 64 bytes Disabled- CommClk+
			ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
		LnkSta:	Speed 5GT/s, Width x4, TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
		DevCap2: Completion Timeout: Range ABCD, TimeoutDis+, LTR-, OBFF Not Supported
		DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis-, LTR-, OBFF Disabled
		LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete-, EqualizationPhase1-
			 EqualizationPhase2-, EqualizationPhase3-, LinkEqualizationRequest-
	Capabilities: [100 v1] Advanced Error Reporting
		UESta:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq+ ACSViol-
		UEMsk:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq+ ACSViol-
		UESvrt:	DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
		CESta:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- NonFatalErr+
		CEMsk:	RxErr- BadTLP- BadDLLP- Rollover- Timeout- NonFatalErr-
		AERCap:	First Error Pointer: 00, GenCap+ CGenEn- ChkCap+ ChkEn-
	Capabilities: [140 v1] Device Serial Number 00-1b-21-ff-ff-53-1e-70
	Capabilities: [150 v1] Alternative Routing-ID Interpretation (ARI)
		ARICap:	MFVC- ACS-, Next Function: 0
		ARICtl:	MFVC- ACS-, Function Group: 0
	Capabilities: [160 v1] Single Root I/O Virtualization (SR-IOV)
		IOVCap:	Migration-, Interrupt Message Number: 000
		IOVCtl:	Enable- Migration- Interrupt- MSE- ARIHierarchy-
		IOVSta:	Migration-
		Initial VFs: 64, Total VFs: 64, Number of VFs: 0, Function Dependency Link: 01
		VF offset: 384, stride: 2, Device ID: 10ed
		Supported Page Size: 00000553, System Page Size: 00000001
		Region 0: Memory at 00000000f7d00000 (64-bit, non-prefetchable)
		Region 3: Memory at 00000000f7c00000 (64-bit, non-prefetchable)
		VF Migration: offset: 00000000, BIR: 0
	Kernel driver in use: vfio-pci
	Kernel modules: ixgbe
```
Within the output, the physical slot is easily identified. On my machine, this card is installed in Slot 2:

```
Physical Slot: 2
```

The documentation is confusing and may lead one to believe the bus address is needed here. However, that is not the case. Armed with this information, you can now exclude the slot from RMRR by creating the following file and replacing `X` in `RMRDS_SlotX` with the physical slot number as shown here:

```
cat <<EOF > exclude.dat
<Conrep> <Section name="RMRDS_Slot2" helptext=".">Endpoints_Excluded</Section> </Conrep>
EOF
```

Now, run `conrep` to update the BIOS accordingly:

```
conrep -l -x conrep_rmrds.xml -f exclude.dat
```
The output will resemble the following:

```
root@aio1:~# conrep -l -x conrep_rmrds.xml -f exclude.dat
conrep 5.2.0.0 - HPE Scripting Toolkit Configuration Replication Program
(c) Copyright 2013,2017 Hewlett Packard Enterprise Development LP

	System Type:			ProLiant DL360e Gen8
	ROM Date   :			08/02/2014
	ROM Family :			P73
	Processor Manufacturer :	Intel

XML System Configuration: conrep_rmrds.xml
Hardware   Configuration: exclude.dat
	Global Restriction: [3.40                            ]                  OK

Loading configuration data from exclude.dat

Conrep Return Code: 0

```
To verify, run the following command:

```
conrep -s -x conrep_rmrds.xml -f verify.dat
```

Read the `verify.dat` file to ensure the slot is excluded:

```
root@aio1:~# cat verify.dat
<?xml version="1.0" encoding="UTF-8"?>
<!--generated by conrep version 5.2.0.0-->
<Conrep version="5.2.0.0" originating_platform="ProLiant DL360e Gen8" originating_family="P73" originating_romdate="08/02/2014" originating_processor_manufacturer="Intel">
  <Section name="RMRDS_Slot1" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot2" helptext=".">Endpoints_Excluded</Section>  <<<<---SHOULD SEE THIS 
  <Section name="RMRDS_Slot3" helptext=".">Endpoints_Included</Section> 
  <Section name="RMRDS_Slot4" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot5" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot6" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot7" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot8" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot9" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot10" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot11" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot12" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot13" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot14" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot15" helptext=".">Endpoints_Included</Section>
  <Section name="RMRDS_Slot16" helptext=".">Endpoints_Included</Section>
</Conrep>
```

Once the change has been made, reboot the system.

# Trying again
Once the system is available, the output of `ovs-vsctl show` should look a little less... error-ish:

```
    Bridge br-provider
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-provider
            Interface br-provider
                type: internal
        Port phy-br-provider
            Interface phy-br-provider
                type: patch
                options: {peer=int-br-provider}
        Port "dpdk-p0"
            Interface "dpdk-p0"
                type: dpdk
                options: {dpdk-devargs="0000:03:00.0"}
```

If the port was previously removed, attempts to create the OVS port should prove successful:

```
ovs-vsctl add-port br-provider dpdk-p0 -- set Interface dpdk-p0 type=dpdk options:dpdk-devargs=0000:03:00.0
```

To verify traffic is traversing the interface, some special network plumbing will need to be constructed. After all, the 10G interface cannot be seen within the OS using tools like `ip` or `ifconfig`, let alone `tcpdump`. The quickest way to observe traffic on the `br-provider` bridge is to mirror all traffic on the bridge to another connected port. This is where veth interfaces come in handy.

Create a veth pair using the following syntax:

```
ip link add tap1 type veth peer name tap0
ip link set tap1 up
ip link set tap0 up
```

Then, connect one end of the veth pair to the `br-provider` bridge while also configuring mirroring:

```
ovs-vsctl add-port br-provider tap0 \
  -- --id=@p get port tap0 \
  -- --id=@m create mirror name=m0 select-all=true output-port=@p \
  -- set bridge br-provider mirrors=@m
```

At this point, a `tcpdump` on the *other* end of the veth pair should reveal traffic:

```
root@aio1:~# tcpdump -i tap1 -ne not stp -c 10
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap1, link-type EN10MB (Ethernet), capture size 262144 bytes
18:31:52.097998 00:0c:29:93:2f:54 > 01:00:5e:00:00:76, ethertype 802.1Q (0x8100), length 297: vlan 5, p 0, ethertype IPv4, 10.5.0.253.5240 > 224.0.0.118.5240: UDP, length 237
18:31:52.098961 00:0c:29:93:2f:54 > 33:33:00:00:01:5a, ethertype 802.1Q (0x8100), length 317: vlan 5, p 0, ethertype IPv6, fe80::20c:29ff:fe93:2f54.5240 > ff02::15a.5240: UDP, length 237
18:31:52.104416 00:0c:29:93:2f:5e > 01:00:5e:00:00:76, ethertype 802.1Q (0x8100), length 297: vlan 50, p 0, ethertype IPv4, 172.31.0.253.5240 > 224.0.0.118.5240: UDP, length 237
18:31:52.106250 00:0c:29:93:2f:5e > 01:00:5e:00:00:76, ethertype 802.1Q (0x8100), length 297: vlan 50, p 0, ethertype IPv4, 172.31.0.1.5240 > 224.0.0.118.5240: UDP, length 237
18:31:52.108869 00:0c:29:93:2f:5e > 33:33:00:00:01:5a, ethertype 802.1Q (0x8100), length 317: vlan 50, p 0, ethertype IPv6, fe80::20c:29ff:fe93:2f5e.5240 > ff02::15a.5240: UDP, length 237
18:31:52.116593 00:0c:29:93:2f:68 > 01:00:5e:00:00:76, ethertype 802.1Q (0x8100), length 297: vlan 50, p 0, ethertype IPv4, 10.50.0.253.5240 > 224.0.0.118.5240: UDP, length 237
18:31:52.120503 00:0c:29:93:2f:68 > 33:33:00:00:01:5a, ethertype 802.1Q (0x8100), length 301: vlan 50, p 0, ethertype IPv6, fe80::20c:29ff:fe93:2f68.5240 > ff02::15a.5240: UDP, length 221
18:31:52.415833 4c:00:82:d2:d5:f8 > 01:00:0c:cc:cc:cd, ethertype 802.1Q (0x8100), length 82: vlan 4089, p 6, LLC, dsap SNAP (0xaa) Individual, ssap SNAP (0xaa) Command, ctrl 0x03: oui Cisco (0x00000c), pid PVST (0x010b), length 56: STP 802.1w, Rapid STP, Flags [Learn, Forward], bridge-id 8ff9.4c:00:82:d2:d6:01.8031, length 56
18:31:52.416748 4c:00:82:d2:d5:f8 > 01:00:0c:cc:cc:cd, ethertype 802.1Q (0x8100), length 82: vlan 2, p 6, LLC, dsap SNAP (0xaa) Individual, ssap SNAP (0xaa) Command, ctrl 0x03: oui Cisco (0x00000c), pid PVST (0x010b), length 56: STP 802.1w, Rapid STP, Flags [Learn, Forward], bridge-id 8002.4c:00:82:d2:d6:01.8031, length 56
18:31:52.417008 4c:00:82:d2:d5:f8 > 01:00:0c:cc:cc:cd, ethertype 802.1Q (0x8100), length 82: vlan 5, p 6, LLC, dsap SNAP (0xaa) Individual, ssap SNAP (0xaa) Command, ctrl 0x03: oui Cisco (0x00000c), pid PVST (0x010b), length 56: STP 802.1w, Rapid STP, Flags [Learn, Forward], bridge-id 8005.4c:00:82:d2:d6:01.8031, length 56
10 packets captured
11 packets received by filter
0 packets dropped by kernel
```

To disable the mirroring, simply run the following command and delete the veth pair:

```
ovs-vsctl clear bridge br-provider mirrors
ip link delete tap0
```

# Summary

The instructions in this guide may not apply only to network interface cards, but also other methods of PCI passthrough using GPUs and other hardware. In my testing, I was able to spin up VMs with the OpenStack API and have them connected to the integration bridge using `dpdkvhostuserclient` ports. I successfully verified connectivity to the VMs from an upstream router. I look forward to kicking the tires on this configuration and performing some benchmarking to see just how much more performance can be squeezed out of the network. If you have any suggestions or feedback, I'm all ears! Hit me up on Twitter at @jimmdenton. 

