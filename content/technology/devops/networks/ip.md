---
slug: fundamentals-the-internet-protocol
guid: https://pasztor.at/blog/fundamentals-the-internet-protocol/
authors:
- janos
categories:
- Technology
date: "2018-12-06T00:00:00Z"
publishDate: "2018-12-06T00:00:00Z"
summary: How do you scale a network to global proportions? The answer is the Internet Protocol. Let's dive into it.
images:
- posts/fundamentals-internet-protocol.png
preview: posts/fundamentals-internet-protocol.jpg
sharing:
facebook: 'Let''s talk about the basic #Internet protocols a little more!'
patreon: Let's talk about the basic Internet protocols a little more!
tags:
- DevOps
- Networks
title: 'Fundamentals: The Internet Protocol'
---

You might have heard of TCP/IP, or at least IP addresses. The IP part of this acronym stands for Internet Protocol. The
Internet Protocol is, besides [Ethernet](/blog/fundamentals-ethernet-explained), one of the most fundamental protocols
of today's global networks.

Ok, wait a second, does this mean [Ethernet, which we previously talked about](/blog/fundamentals-ethernet-explained),
is obsolete and IP is the new thing?

Well, not exactly. It is true that Ethernet can only be used between devices on the same network, but IP does not
replace it. Instead, IP is transported *on top* of Ethernet.

Let's start with the basics: we want to assign each of our devices a unique address. Preferably globally unique so that
the address can be used on a global level.

We do this by taking a very large number block, say zero to roughly four billion. We can store this number on 32 bits of
space. You may notice that four billion addresses are not even enough for every human on Earth, we'll get to that later.

Now, we carve up this vast number space and give each provider a chunk of it, say 0-1023 for one network provider,
1024-2047 for the next, and so on.

Each network provider can assign the IP addresses to their devices as they see fit. Early on this would be done by hand,
later this would be automated using a protocol called [DHCP]().

So each device has a unique number. Let's call it by its proper name: an IP address. How do we get a data packet from
a computer in the network of provider A to a computer in the network of provider B?

For our example these two providers must agree to exchange data. They deploy a common networking device called a
**router** that is connected to *both networks*. Additionally they will feed the router the information on which network
can be accessed over which cable. This is called a **routing table**.

## Sending data on a local network

When our computer on network A wants to send some data to another computer within the same network, the situation is
simple. It too has a *routing table* and knows which network block it is on. Say, 0-1023.

Similar to Ethernet frames, IP packets also have source and destination addresses in them. Our computer now writes its
own IP address into the *source* field and the other computers IP address into the *destination* field.

However, it can't just simply blindly fire this IP packet onto the local network. It needs to package the IP packet into
an ethernet frame. For that it needs the *MAC address* of the target computer.

The MAC address is obtained using the **Address Resolution Protocol** (ARP) or the **Neighbor Discovery Protocol**
(NDP), depending on which IP version we use.

Once the MAC address is found out, our computer creates an Ethernet frame with the target MAC address, puts the IP 
packet inside and sends it off on the local network.

## Sending data outside the local network

Now here comes the tricky part. What if the target computer is on network B and not on the local network?

As before, our computer creates the IP packet, but also realizes that the target computer is not on the local network.
In its routing table it will find an entry for **default gateway**: the IP address of the router. The default gateway 
entry means that any packet that doesn't have a more specific rule in the routing table, will be sent there.

So, instead of using ARP or NDP to find out the MAC address of the *target* computer, instead the MAC address of the 
*router* is looked up. Our computer creates an Ethernet frame with the *router* as a destination and puts the IP packet
inside.

The router then unpacks the Ethernet frame and looks at the IP packet inside. Using its *own routing table* it
determines where to send the packet next. Since it is directly connected to network B directly, it can do an ARP or
NDP lookup on network B to figure out the MAC address of the target computer, pack up the IP packet in a new Ethernet
frame and send it out over the wire on network B.

So in essence, there is a separate Ethernet frame for each network the IP packet travels through and each router or
device evaluates the IP packet separately.

## IP address exhaustion

Now, as I've indicated before, around 4 billion addresses are not enough even if just every human on Earth
has one device. The older, version 4 of the internet protocol, unfortunately uses 32 bits for addresses, so we are
rapidly reaching the point of exhausting all available IP addresses.

The newer, version 6, of the internet protocol uses 128 bit addresses, so every grain of sand in the Sahara could get an
address, but the adoption has been slower than expected. The reason for this is partially die to the fact that IPv6
brings in a *lot* of new features, which makes it a bit of a learning curve to deploy.

To work around IP exhaustion, Network Address Translation (NAT) was invented. Home routers regularly assign devices
IP addresses from one of the so-called *private* IP ranges (`192.168.`, `172.24`, etc) and then, when a packet needs to
go into the outside world, *change the source IP address* of the packet.

They also *remember* the packet, so if a response is received, it can be sent back to the private IP address it came
from. This is also known as masquerading.

Since the IP exhaustion has reached big providers, they sometimes even deploy NAT on a massive scale, commonly 
referred to as Carrier-grade-NAT or CGN. 

## Scaling

I've mentioned that IP works on a global scale. As you might imagine, coordinating the routing tables of each and every
router in the world is a massive effort.

That's why this isn't done manually, but by using so-called **routing protocols** such as the **Border Gateway
Protocol** or BGP. Using BGP providers announce to each other which routes are available through their connections and
the routers automatically update their routing tables.

This, of course, [can cause problems](https://blog.cloudflare.com/the-story-of-two-outages/), which is why internet
providers need to make sure they only accept routing announcements from parties they belong to. But that is a story
for another article.