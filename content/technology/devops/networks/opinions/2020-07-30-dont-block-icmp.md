---
title: Don’t Block ICMP
slug: dont-block-icmp
guid: https://pasztor.at/blog/dont-block-icmp/
summary: “Don't ping my server!” — is the sentiment among many sysadmins, and usually leads to a full-on ICMP blocking. But, it's a terrible idea. Here's why.
authors:
- janos
categories:
- Technology
images:
- posts/dont-block-icmp/social.png
tags:
- Cloud
- DevOps
- Networks
date: "2020-07-30T00:00:00Z"
publishDate: "2020-07-30T00:00:00Z"
---

If you have spent any length of time in IT operations, you know the sentiment: <q>“Don’t ping my server!”</q> Some would say it’s security by obscurity, but it does make sense on some level. You are presenting a smaller attack surface to automated tools.

However, over the years, I have seen a staggering amount of firewalls that simply achieve this by blocking ICMP wholesale. This is unfortunate since ICMP is useful for more than just merely sending and receiving pings.

## What is ICMP?

The Internet Control Message Protocol, as the name implies, is a way to send control messages between two hosts on the Internet. One of these control messages is the infamous ping, or as it is officially called echo request and echo response.

ICMP comes with two numeric fields: the type and the code field. These together make up the message type in the ICMP packet. For example, type 0 and code 0 together are an echo (ping) response. Type 3 is a message type indicating that the destination cannot be reached, the different codes indicate the reason why. You can find the details on these codes [on Wikipedia](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol#Control_messages).

## So, why is ICMP blocking bad?

If you look at the list of types and codes on Wikipedia, you may see a few useful ones. Let's take a look at what happens when you block these types?

Take type 3: `destination unreachable`. Let's say you set up your server to block incoming ICMP as a whole or specifically type 3 messages, and you are trying to connect a remote server. If this remote server blocks your connection, two things can happen. If you are trying to open a TCP connection *and* the remote firewall is set up correctly, you will receive a TCP reset without problems. If, however, you are trying to send UDP packets *or* the remote server is not set up to respond with a TCP reset, you may, instead, receive an ICMP destination unreachable. Receiving an ICMP destination unreachable means that your software will hang until the timeout is exceeded.

But that's not the worst that can happen. Let's take type 3 code 4: `fragmentation required`. IP fragmentation is a tricky topic, but let me give you a brief run-down. Your average connection on the Internet can transport 1500 bytes in a single IP packet. This number is called the MTU, the maximum transmission unit. However, some connections  can transport less than that. Your average VPN, for example, can only carry 1492 bytes.

The default way to deal with this is splitting the packet into two at the router on the edge of the lower MTU connection. However, if the packet has the &ldquo;don't fragment&rdquo; bit set, this router sends back an ICMP packet with the `fragmentation required` type. If your firewall blocks this ICMP packet on your server, the connection will be stuck in limbo.

## Why can't my cloud provider fix this?

There was an interesting question regarding this: why don't cloud providers handle this automatically? The answer is flexibility.

In some cases, you want to use your public IP’s for internal services. In this case, you really, really want to block *everything*. You want your server’s IP to respond with *nothing* at all. In this case, you need to be able to block ICMP along with everything else.

## What should I do?

If you want to prevent pings to your server, you should block ICMP types `0` and `8`. You are also on the safe side blocking timestamp packages (type `13` and `14`) and extended echo requests and responses (type `42` and `43`). 