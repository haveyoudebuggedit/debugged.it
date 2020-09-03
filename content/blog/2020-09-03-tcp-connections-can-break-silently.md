---
title: TCP connections can break silently
slug: tcp-connections-can-break-silently
summary: "Are you thinking of using Microservices? Here’s something you may not know: TCP connections can break entirely silently."
authors:
- janos
categories: blog
images:
- posts/tcp-connections-can-break-silently/social.png
tags:
- Software Development
- DevOps
- Networks
date: "2020-09-03T00:00:00Z"
publishDate: "2020-09-03T00:00:00Z"
---

Here’s a little thing that may have big consequences, especially when it comes to microservices: your TCP connections can break silently, without you even knowing about it.

TCP connections silently breaking can cause your microservices to hang indefinitely. What’s worse, this may only happen in production, and I’ll explain in this post why. If you are experienced in writing microservices you may have worked around that by implementing a [circuit breaker](https://martinfowler.com/bliki/CircuitBreaker.html) in your application. However, if you are just getting started with microservices you may want to keep reading.

*“Wait a minute! ”* &mdash; you may think &mdash; *“ Isn’t TCP supposed to be reliable?”* Yes, you are right. TCP guarantees any submitted data to be either reliably transmitted, or you will receive an error. However, if you transmit no data over a connection TCP (on default settings) does not check if the connection is still alive.

How does this check work when data is sent, you ask? Every so often the receiving party sends an *acknowledgement* packet to the sending party for the data it has received. If an acknowledgement is not received within 3 seconds the TCP packet is *retransmitted*. If no data is sent, no acknowledgement is sent either. In other words, there is no packets are traveling between the two parties.

Let’s apply this to microservices: service A sends a HTTP request to services B. Services A then waits for a response. During this wait period neither A, or B, send any data over the TCP connection. When service B is finally ready with processing the request it will send the response to service A and the connection will be closed.

So far so good. However, what happens if the machine service B runs on suddenly hits a kernel panic? In other words the network link stays up on a hardware level, but the software doesn’t respond at all. In this case service A  will consider the TCP connection to be alive up until the point where it hits a timeout.

The default for this timeout is 10 minutes. Service A may hang for 10 minutes. Other services that rely on service A may also hang for that amount of time.

{{% warning %}}
**Be careful!** In a development environment you may not see this happening if all your services run on the same machine. TCP connections always give you a connection closed error if the other party crashes. 
{{% /warning %}}

So, what can you do? First of all, don’t use microservices if you don’t have to. I know, it’s the cool thing to do, but microservices bring *more* complexity due to the network in-between, not less. Other than that, you can implement the aforementioned [circuit breaker pattern](https://martinfowler.com/bliki/CircuitBreaker.html), tune TCP timeouts, and turn on [TCP keepalives](https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html).