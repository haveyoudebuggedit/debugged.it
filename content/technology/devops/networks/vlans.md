---
slug: fundamentals-vlans
guid: https://pasztor.at/blog/fundamentals-vlans/
authors:
- janos
categories:
- Technology
date: "2018-11-29T00:00:00Z"
publishDate: "2018-11-29T00:00:00Z"
summary: How do you run multiple networks over the same physical network? How do virtual LANs work?
images:
- posts/vlans-explained.png
preview: posts/vlans-explained.jpg
tags:
- DevOps
- Networks
title: 'Fundamentals: VLANs explained'
---

In our [previous article](/blog/fundamentals-ethernet-explained) we've talked about how Ethernet works. Ethernet is the
most fundamental protocol we use today, but it was designed for one network, one physical cable. However, as the
networks became more and more complex, laying out a physical cable for each and every network became unrealistic.

And people wanted separate networks for separate roles, for security and ease of management, so you couldn't just simply
put everyone on the same network.

Long story short, there was a need for running multiple networks on the same physical cabling. And that's where the
802.1Q standard comes into play, commonly referred to as VLANs (Virtual LANs).

From the [previous article](/blog/fundamentals-ethernet-explained) you may remember that the Ethernet frame has a field
called **ethertype** which describes what the contents of that Ethernet frame are.

Now, 802.1Q employs a little trick. It sets the Ethertype to `0x8100`, indicating that this is an 802.1Q tagged frame. 
It then reserves the first 4 bytes of the payload.

The first 2 bytes of the payload indicate the **VLAN ID**, or in other words the **VLAN tag**. This tag tells the 
devices on the network which VLAN the Ethernet frame belongs to.

The next 2 bytes of the payload then have *another* Ethertype field, which now indicates what the contents of the 802.1Q
tagged frame are.

![](posts/ethernet-vlan.svg "The Ethernet II frame structure with an embedded 802.1Q header")

## VLANs in practice

When it comes to the practical use of VLANs we usually talk about **tagged** and **untagged** connections. 
When we talk about *tagged*, we mean that the device is receiving packets with a a 802.11Q VLAN tag. This way even
a normal computer can be in multiple networks over the same physical connection.

In contrast, when we talk about *untagged*, we mean that the device receives packets *without* a 802.11Q VLAN tag. 
You can, of course, run one VLAN untagged to the device and the rest tagged over the same wire. Switches can be
configured to *translate* between tagged and untagged packets as needed.

Let's look at an example. On Ubuntu Linux we can install the `vlan` package to get a hold of the user space programs
that manage VLANs. Once that's done, we can do something like this:

```
vconfig add eth0 2
```

This command adds the tagged VLAN `2` to the `eth0` interface. (The untagged VLAN does not need to be added and the
computer does not know which VLAN the untagged one is.)

We can then pull the interface up:

```
ip addr add 192.168.0.1/24 dev eth0.2
ip link set up eth0.2
```

If we want to make this configuration permanent, we can do so in `/etc/network/interfaces`:

```
auto eth0.2
iface eth0.2 inet static
    address 192.168.0.1
    netmask 255.255.255.0
    vlan-raw-device eth0
``` 

And after an `ifup eth0.2` we should have the VLAN up and running!