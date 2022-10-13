---
title: "Neutron Dynamic Routing - What it is (and isn't)"
layout: post
date: 2022-10-12
image: /assets/images/2022-10-12-openstack-bgp-speaker/jeff-bridges-what.gif
headerImage: true
tag:
- neutron
- openstack
- bgp
category: blog
blog: true
author: jamesdenton
description: "An explanation of what BGP Speaker is and isn't"
---

To understand OpenStack Neutron's Dynamic Routing feature, you must first understand what BGP Speaker is... and what it isn't.
<!--more-->

Recent workshops with a customer made it very clear to me that Neutron's Dynamic Routing feature leaves a lot on the table, and likely isn't a good fit for many of the environments that would look at using it. That doesn't mean it isn't useful, though.

Before jumping too far into Neutron Dynamic Routing and it's core function, advertising tenant networks, let's revisit Neutron's logical network designs.

## Tenant Networking

Whether you're using ML2/LXB (Linux Bridge), ML2/OVS (Open vSwitch), or ML2/OVN, the *logical* network topology for tenant networking looks relatively the same. It's composed of:

- An external provider network
- A virtual router
- One or more tenant networks

On paper, it looks something like this:

![Standard Tenant Networking](/assets/images/2022-10-12-openstack-bgp-speaker/standard_tenant_networking.png)

Tenant networks are not reachable by default. The virtual router can source NAT (SNAT) outbound traffic from instances in tenant networks to allow connectivity to external networks or the Internet. Inbound traffic in this scenario is not possible without the use of Floating IPs. Floating IPs, in turn, are sourced from the **external provider network**. Your standard Neutron tenant network topology looks something like this:

![Floating Tenant Networking](/assets/images/2022-10-12-openstack-bgp-speaker/floating_tenant_networking.png)

To reach tenant networks directly and bypass the use of floating IPs, one *could* implement a static route on the provider network gateway device and redistribute that route upstream. In fact, we've done this for many years as far back as the Grizzly release of OpenStack, when Neutron (neé Quantum) was in its infancy. Where this falls apart, though, is in the ***self-servicing*** of tenant networking. Tenants can't (or shouldn't) access that provider gateway device and would not be able to add that static route. 

## Neutron Dynamic Routing

The obvious solution is to implement some sort of dynamic routing mechanism to allow tenants to advertise their tenant network(s) upstream with no involvement from the network administrator. Neutron provides this capability with a combination of **Neutron Dynamic Routing**, **Subnet Pools**, and **Address Scopes**.

Neutron Dynamic Routing provides a service known as **BGP Speaker** that peers with external routers to advertise the tenant networks using BGP. Subnet pools and address scopes are used together to avoid overlapping subnets, especially when advertising to a given peer.

Where the misunderstanding appears to sneak in is *how* and *where* the advertisements occur. It's fairly common practice to have two routers directly connected to one another to exchange routes, like so:

![BGP](/assets/images/2022-10-12-openstack-bgp-speaker/bgp.png)

One might assume, then, that the Neutron router would peer with the provider network router in this fashion. They'd be wrong!

![No BGP For you!](/assets/images/2022-10-12-openstack-bgp-speaker/soup.jpg){:width="350"}

That's where **BGP Speaker** comes into play. The BGP Speaker is a *control plane* service that advertises tenant network(s) on behalf of the tenant router. The BGP Speaker peers with the provider network router and advertises the tenant network with a next hop of the tenant router, like so:

![BGP Speaker!](/assets/images/2022-10-12-openstack-bgp-speaker/speaker.png)

The BGP Speaker is not a router. It is not a route reflector. It does not accept BGP routes from other speakers or routers. It. Only. Speaks. BGP. And, it does this from the control plane or network node hosting the BGP "dragent". What that means in practice is that the controller or network node hosting the agent needs L3 connectivity to the provider network gateway device: either the WAN, the LAN, or some other interface to peer on. This requirement is not ideal in many environments and could be a deal breaker in others. 

## Summary

The documentation upstream for [Neutron Dynamic Routing](https://docs.openstack.org/neutron/latest/admin/config-bgp-dynamic-routing.html) has some pretty good diagrams and goes into further detail than what I've described here. The BGP speaker can even advertise floating IPs, though I'm not sure how this makes sense if the provider router is locally connected. However, I'm sure there's a use case I haven't considered. There have been attempts to implement BGP at the Neutron router itself, as seen in [RFE](https://bugs.launchpad.net/neutron/+bug/1921461), but it has not really made much traction since late 2021. This functionality would mirror something I've seen in NSX and other (legacy) cases, but might result in too much overhead; especially when hundreds of routers are involved.

---
If you have some thoughts or comments on this post, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
