---
title: "Speeding Up Multi-Node All-In-One (MNAIO) OpenStack-Ansible Deployments"
layout: post
date: 2019-01-03
image: /assets/images/2019-01-03-nvme-homelab/SpinningWheelOfDeath.jpg
headerImage: true
tag:
- nvme
- hp
- proliant
- openstack-ansible
category: blog
blog: true
author: jamesdenton
description: "Speeding Up Multi-Node All-In-One (MNAIO) OpenStack-Ansible Deployments w/ NVMe"
---

The OpenStack-Ansible project provides an [All-In-One](https://docs.openstack.org/openstack-ansible/latest/user/aio/quickstart.html) (AIO) deployment playbook for developers that creates a functional OpenStack cloud on a single host and is perfect for testing most basic functions. For advanced development and testing, however, a [Multi-Node AIO](https://github.com/openstack/openstack-ansible-ops/tree/master/multi-node-aio) deployment can be performed that deploys several virtual machines on a single bare-metal node to closely replicate a production deployment.

<!--more-->
My beef with the MNAIO deployment up until now has been _time_. In a stock MNAIO deployment, twelve different virtual machines are built and bootstrapped using PXE. The Ubuntu 16.04 LTS operating system is installed with the Ubuntu installer. Once the operating system is installed, a full OpenStack deployment occurs with the following breakdown:

<center>

| Service       | Count |
|---------------|:-----:|
| Infra         | 3     |
| Swift         | 3     |
| Cinder        | 2     |
| Compute       | 2     |
| Load Balancer | 1     |
| Logging       | 1     |

</center>

Needless to say, there are a lot of actions that need to occur to complete the cloud deployment. Time is money, as they say. In this case, spending a little money upfront can give you back a whole lot of time along the way.
 
# Getting started

For this comparison, I'm working with the following bare-metal node in a home lab setting:

- HP Proliant DL360p G8
- 2x Intel E5-2667v2 3.3Ghz processors
- 192GB RAM
- Micron M500 480GB SSD Hard Drive (sda)
- HGST 7200rpm 1TB SATA Hard Drive (sdb)
- Ubuntu 16.04 LTS Operating System

For an MNAIO deployment, the `sdb` drive is used by default as the VM data disk. Virtual machine hard drives are stored there. As one might expect, the speed of that drive plays a big part in the duration of the deployment process.

To deploy a Multi-Node All-In-One OpenStack Cloud based on OpenStack-Ansible, simply clone the GitHub repository shown here:

```
git clone https://github.com/openstack/openstack-ansible-ops/
```

Navigate to the `multi-node-aio` directory and execute `build.sh`:

```
cd openstack-ansible-ops/multi-node-aio/
./build.sh
```

---
**NOTE**

Many different configurables are available, just check the README.

---

# Go get coffee. Maybe two.

After first experiencing a MNAIO deployment and waiting for it to finish, I made some modifications to the playbooks and scripts to document start and end times so that I'd have a better idea as to how long it took for VMs to deployed and for certain playbooks to execute. With the 1TB spinning SATA disk, the results are broken down as follows:

<center>

|          | build.sh | <span style="color:#D3D3D3">VM Setup</span> | Setup Hosts | Setup Infrastructure | Setup OpenStack | Total   |
|----------|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| Duration | <span style="color:red">51:38</span>    | <span style="color: #D3D3D3">36:43</span>   | 55:03       | 56:41                | 1:39:27         | <span style="color:red">4:22:49</span> |

</center>

The `build.sh` script is executed by the user and installs packages on the bare-metal node to support the hosting of virtual machines. By default, twelve virtual machines are setup and deployed with the Ubuntu OS using a preseed over PXE. Once deployed, the script will determine they are online and bootstrap the OpenStack deployment. When the bootstrap is complete, the script will exit while OpenStack continues to be deployed from the **infra1** virtual machine to the rest of the virtual machines.

Of the 51 minutes it took to execute `build.sh`, it took over 36 minutes for the virtual machines to become available and recognized by the script. Once the OpenStack deployment was underway, the first two playbooks, `setup-hosts.yml` and `setup-infrastructure.yml`, took nearly two hours to complete. The bulk of the OpenStack installation took 1 hour and 39 minutes, for a total execution time of 4 hours and 23 minutes.


# Bring on the speed

Previous experience with NVMe benchmarking led me to immediately consider upgrading the 1TB SATA spinning disk to a 1TB NVMe disk. I knew to expect a decent improvement, especially since all benchmarks show NVMe performance spanking even SATA-based SSDs, let alone spinners. 

For testing, I picked up the following:

- HP EX920 M.2 1TB x4 PCIe NVMe drive ([Amazon](https://amzn.to/2GVDfMn))
- Mailiya M.2 PCIe to PCIe 3.0 x4 Adapter ([Amazon](https://amzn.to/2LNVZMJ))

Wiping the host and starting a fresh deployment with an NVMe data disk showed some improvement: 

<center>

|          | build.sh | <span style="color:#D3D3D3">VM Setup</span> | Setup Hosts | Setup Infrastructure | Setup OpenStack | Total   |
|----------|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| Duration | <span style="color:green">31:11</span> | <span style="color:#D3D3D3">23:46</span>   | 32:05       | 51:33                | 1:19:02         | <span style="color:green">3:13:51</span> |

</center>

Of the now 31 minutes it took to execute `build.sh`, 23 minutes was spent deploying the virtual machines. Over 30 minutes was shaved off the execution of the first two OSA playbooks, `setup-hosts.yml` and `setup-infrastructure.yml`, and 20 minutes was saved on the `setup-openstack` playbook. Total execution time dropped nearly 70 minutes to 3 hours and 14 minutes.

Further improvement could be seen by changing the CPU scaling governor from **ondemand** to **performance**, which resulted in an additional 30 minutes saved for a total execution time of about 2 hours and 45 minutes. For a home lab scenario, the increased noise and heat generated by the server in **performance** mode wasn't really worth the time saved, in my opinion.

I performed a second test with another NVMe drive that claimed up to 3500MB/sec reads and 3000MB/sec writes, compared to the 3200MBps/1800MBps rating of the HP drive:

- ADATA XPG SX8200 Pro 1TB 3D NAND NVMe Gen3x4 PCIe M.2 ([Amazon](https://amzn.to/2RsX7KZ))

The proof is in the pudding, as they say, and the results can be seen here:

<center>

|          | build.sh | <span style="color:#D3D3D3">VM Setup</span> | Setup Hosts | Setup Infrastructure | Setup OpenStack | Total   |
|----------|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| Duration | <span style="color:green">35:46</span> | <span style="color:#D3D3D3">20:08</span>   | 33:18       | 47:17                | 1:17:54         | <span style="color:green">3:14:15</span> |

</center>

There's a margin of error here that says the differences between these two drives is negligible. My advice? Save your money for this use-case and stick with the HP EX920 or something similar.

# OnMetal

For grins, I thought it would be interesting to compare the performance of my server with an NVMe disk to a [Rackspace OnMetal I/O v2](https://www.rackspace.com/en-us/cloud/servers/onmetal) node mentioned in the MNAIO [docs](https://github.com/busterswt/openstack-ansible-ops/tree/master/multi-node-aio). An OnMetal I/O v2 node is equipped with dual 2.6Ghz 10-core Intel E5-2660 v3 processors, 128GB RAM, and dual 1.6 TB PCIe flash cards of the LSI Nytro Warpdrive BLP4-1600 variety. 

The performance was better than I expected:

<center>

|          | build.sh | <span style="color:#D3D3D3">VM Setup</span> | Setup Hosts | Setup Infrastructure | Setup OpenStack | Total   |
|----------|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| Duration | <span style="color:green">27:21</span> | <span style="color:#D3D3D3">18:57</span>   | 25:44       | 24:36                | 53:36         | <span style="color:green">2:11:17</span> |

</center>

With an OnMetal I/O v2 node, the total time dropped an additional hour. There are some possible factors that play into this substantial decrease in time beyond storage, such as network bandwidth, that are difficult to determine. But needless to say, I'll consider the OnMetal servers if I really need to hit some deadlines and the price is right.

# Summary

The drop in NVMe storage pricing makes it an attractive option for homelabbers like myself who value the time savings and are willing to pay a fair price for it. I'm all about multitasking, but delivering results is even more important to me. Being able to save at least an hour a day, maybe two, means this NVMe investment will pay for itself in no time at all.

See you in #openstack-ansible on FreeNode IRC!