---
slug: fundamentals-ethernet-explained
guid: https://pasztor.at/blog/fundamentals-ethernet-explained/
authors:
- janos
categories:
- Technology
date: "2018-11-28T00:00:00Z"
publishDate: "2018-11-28T00:00:00Z"
summary: Ethernet is one of the most fundamental protocols underpinning todays internet. It is so fundamental that we often take it as granted and don't even think about it.
images:
- posts/ethernet-explained.png
preview: posts/ethernet-explained.jpg
tags:
- DevOps
- Networks
title: 'Fundamentals: Ethernet explained'
---

{{% tip %}}
**Did you know?** This article is also available [as an animated video](https://www.youtube.com/watch?v=5u52wbqBgEY).
{{% /tip %}}

## A Little History

Back in the early days before the Internet computer networks were much smaller and focussed more on local connectivity. Ethernet, or more accurately Ethernet II, was but one of the many protocols people used to connect computers into a local network.  Other contenders included, for example, IPX, which later died out due to it's inability to scale.

In those early days network hardware was comparatively expensive and the system we have today where each computer is connected to a router with a dedicated cable, or even over the air, would have been unimaginable. Most commonly, computers were connected to a single, very long coaxial cable that had to be terminated with a plug at the end.

Yes, you read that right. All computers on a single cable. When a computer wanted to send a data packet over the network
it had to first &ldquo;shout&rdquo; a sequence of bits (preamble) on the wire to indicate that it would send data. If
two computers did this at the same time you had a conflict which both computers would detect. They would then stop
sending and wait a random amount of time before trying again. This process repeated until one of the computers managed
to get their preamble out.

All computers on the network would listen for the preample and if they &ldquo;heard&rdquo; it they would know that
someone was sending data over the network and would not attempt to send out a preamble until they encountered a 
so-called interframe gap, 12 bytes (*octets*) of &ldquo;silence&rdquo;.

## The Modern Ethernet

![](posts/ethernet.svg "The Ethernet II frame structure")

Decades have passed and now networking hardware is available in abundance. Ethernet chips cost next to nothing, we have
dedicated cables for all our machines and even 10 gigabit interfaces are no longer a rarity. IPX and it's cousins have
all but died out and most of our networks are based on almost the same Ethernet protocol we used almost four decades
ago.

As mentioned in the previous segment, the first thing a device sends out is a 7 byte (or octet in old money) **preamble**. This preamble consists of 7 times the sequence `10101010`, or `0x55` in hexadecimal notation.

The preamble is followed by the **start frame delimiter**, which consisted of the sequence `10101011` (`0xD5`). The SFD
indicated that the delimiter is finished and that data would now commence.

Now, the devices on the network would need to know who sent the packet and who it was intended for. This was done by
adding so-called MAC or hardware addresses. You might have seen them, they look like this: `42:cb:c0:14:9b:0e`
MAC addresses are 6 byte (octet) addresses that are *unique to the network chip*. The first half of the address space
is *assigned to a network hardware vendor* and the second half can then be freely assigned. This made sure that all
MAC addresses are (hopefully) globally unique. The MAC addresses also have two special bits included in the vendor part
to indicate multicast packets (sent to more than one destination) and locally administered MACs to create a range for
&ldquo;virtual&rdquo; MAC addresses (for example for virtual machines).

In the Ethernet frame the **destination** and **source MACs** are added after the SFD so all participants know who is 
sending the packet to whom.

The next bit in the packet is a bit complicated. It is called the **ethertype**. Originally it was used to indicate the
length of the payload, but it was soon discovered that there is a need to indicate what kind of data the packet is
transporting. Therefore, the ethertype values *under 1500* indicate the length, whereas ethertype values *over 1536*
indicate the payload type. For example the value of `0x0800` indicates that an IPv4 packet is being transported.

After the ethertype we can finally send the **payload**. The payload can be anywhere between 46 and 1500 bytes, or with
*jumbo frames* even up to 9000 bytes. This is also known as the Maximum Transmission Unit or MTU size of a connection.

The payload is followed by a **Frame Check Sequence**, which is basically a CRC32 checksum over the payload. It ensures
that the data wasn't corrupted in transit.

Finally the ethernet frame is closed with a 12 byte **Interpacket Gap**, or 12 bytes of &ldquo;silence&rdquo;.

## How it is used

Ethernet works great on the local network. The problem comes when the target device is not on the local network, but 
behind one or more routers. Ethernet can't deal with that, but other protocols like the Internet Protocol or IPX can.
These protocols are designed to be *encapsulated* in Ethernet.

In laymans terms this means that an Ethernet frame can *transport* an IP packet inside of it. When it reaches a router,
the router will unpack it and forward the IP packet. But that is a topic for another article.