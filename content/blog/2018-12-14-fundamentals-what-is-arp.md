---
slug: fundamentals-what-is-arp
guid: https://pasztor.at/blog/fundamentals-what-is-arp/
authors:
- janos
categories:
- Technology
date: "2018-12-14T00:00:00Z"
publishDate: "2018-12-14T00:00:00Z"
summary: How does a computer know what MAC address belongs to an IP? How does ARP
  work?
images:
- posts/fundamentals-arp.png
preview: posts/fundamentals-arp.jpg
tags:
- DevOps
- Networks
title: 'Fundamentals: The Address Resolution Protocol'
---

In two previous articles we discussed how [Ethernet](/blog/fundamentals-ethernet-explained) and
[the Internet Protocol](/blog/fundamentals-the-internet-protocol) work, but we've glossed about a very important aspect
of the two: how does a computer or a router know which MAC address to send an IP packet to?

In case of IP version 4 the answer is the Address Resolution Protocol, whereas in IPv6 it's called Neighbor Discovery 
Protocol. This article will detail the Address Resolution Protocol.

ARP builds on top of Ethernet itself, the communication is done by sending single Ethernet frames. As a first
example, let's look at a lookup for an IP address.

Let's say a router wants to know the MAC address of `192.168.0.2`. In this case it would send out an ARP Request packet
that translates to *Who has 192.168.0.2?*. The source address of this Ethernet frame will be the MAC address of the 
device sending the request, the destination will be the broadcast MAC of `FF:FF:FF:FF:FF:FF`. This address means that
switches on the network will forward the packet to all devices.

However, the ARP protocol has its own source and destination fields, but they are used a little differently. For an
ARP request the destination MAC in the ARP packet will be `00:00:00:00:00:00` instead of `FF:FF:FF:FF:FF:FF`. The
all-zero mac is equivalent to *unspecified*.

Is this confusing? Yepp, it is. The source and destination MAC are present in both the Ethernet layer and the ARP
packet. And yes, they may be different. But more on that later.

## ARP caching

As you might imagine, sending an ARP request for every single packet is quite expensive. That's why all devices cache
ARP responses, and they do that for quite a long time. If you run on Linux you can check the ARP table with the 
following command:

```
ip neigh
```

## Gratuitous ARP

Sometimes ARP requests and replies can be sent gratuitously, or in other words, out of the normal order. There are 
many reasons to do this, but there is one important aspect to keep in mind: if a device / computer receives an ARP
response packet, the results are **cached**. And what's even more peculiar, the receiving device doesn't care if the
sending device and the information contained within the ARP packet are in any relation to each other.

In other words, *any* device can send an ARP packet to change the ARP table of other devices. Sounds like a source of
trouble? Well, it is. However, there are legitimate uses too, so you can't just change this behavior.

### Clustering

Let's talk a little bit about legitimate uses of Gratuitous ARP. First, let's look at **clustering**. It is quite common
pattern when building clusters to create a so-called virtual IP or VIP. This VIP is shared between two or more machines
in a way that at any given time only one device &ldquo;has&rdquo; the IP.

However, when a failover is desired, a different machine takes on the VIP in addition to its original IP. The router,
however, still thinks the IP is on the old host and it will stay like that for a while. To fix this issue, the new
owner of the IP will send a gratuitous ARP packet to the router, triggering it to update its ARP table.

### Switches

Modern switches (networking equipment) keep track of which MAC address is on which port. Sending an ARP packet can 
be used to inform the switch that a given MAC address is on a given port. This is especially important when virtual
machines are moved between hosts.

### Preloading ARP tables

When network links come up (e.g. when a cable is plugged in) modern network stacks will send a gratuitous ARP packet
to preload the neighboring machines' ARP tables with the IP addresses on that host.

### IP address usage checking

Gratuitous ARP can be used to check if an IP address is already in use on a network. This is often done if a 
network works with self-assigned IPs. It is also useful to detect IP address conflicts. If a machine receives an ARP
packet with its own IP but a different MAC, it knows that the IP is in use elsewhere. 

## ARP spoofing

I mentioned previously that the MAC address is present both in the Ethernet frame and the ARP packet. The sad state
of affairs is that the two are in no relation to each other. *Any* host can send an ARP request or reply to *any* other
host with *any* MAC address on the ARP layer. For example, we could poison the ARP cache of a target machine like
this: 

```
nping \
  --source-mac <attackers MAC> \
  --dest-mac <victim MAC>
  --arp
  --arp-type ARP-reply \
  --arp-sender-mac <attackers MAC> \
  --arp-sender-ip <router IP> \
  --arp-target-mac <victim MAC> \
  --arp-target-ip <victim IP> \
  <victim IP>
```

This will effectively poison the victims ARP cache so that they send any traffic they intended to send the router
to your machine.

You could even go one step further and, instead of specifying your own MAC, send the traffic elsewhere. Yes, ARP is
basically a free-for-all. 

## Summary

ARP is a very basic protocol, but due to its open nature it lets you do all kinds of shenanigans. When ARP fails, 
it is quite hard to debug because its functionality is taken for granted.

## Sources

- [https://en.wikipedia.org/wiki/Address_Resolution_Protocol](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)
- [https://wiki.wireshark.org/Gratuitous_ARP](https://wiki.wireshark.org/Gratuitous_ARP)