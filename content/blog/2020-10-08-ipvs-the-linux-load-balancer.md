---
title: "IPVS: The Linux Load Balancer (Deep Dive)"
slug: ipvs-the-linux-load-balancer
summary: "Did you know Linux has a built-in network load balancer? Did you know it is used by Kubernetes? Let’s explore!"
authors:
- janos
categories:
- Technology
images:
- /posts/ipvs-the-linux-load-balancer/social.png
tags:
- Networks
- DevOps
date: "2020-10-08T23:00:00Z"
publishDate: "2020-10-08T23:00:00Z"
---

It’s been part of the Linux kernel forever: IPVS, the IP Virtual Server. You’d be forgiven for now knowing about it, as it is unlikely to come across your radar unless you are in need of a high performance network load balancer.

This article will explain the theory behind, and the practical operations of the IPVS load balancer included in the Linux kernel.

IPVS is part of netfilter, the packet filtering and translation framework built into the Linux kernel. Netfilter is a vast topic to explain, so we won’t go into any details here. Suffice it to say, IPVS is a very powerful load balancer that is in use, for example, in Kubernetes.

## A (tech) manager’s overview

The tricky part about load balancing is handling the amount of traffic. IPVS offers three operation modes, ranging from a simple, but low throughput NAT mode to the complex but scalable IPIP mode.  
 
![](/posts/ipvs-the-linux-load-balancer/nat-01.svg "IPVS in NAT mode")
 
The simplest mode of operations is the **NAT/Masquerade mode**. The load balancer listens on a dedicated IP address (VIP; or virtual IP) and when a connection request or a packet belonging to an existing connection come in, it changes the destination IP of the packet to one of its backend servers. The packet is then forwarded to that backend with the new IP address. The main benefit of this mode of operation is its simplicity, but it comes at a cost: all the return packets also need to pass through the load balancer, so it can change the source IP address back to the VIP.

![](/posts/ipvs-the-linux-load-balancer/dr-01.svg "IPVS in Direct Routing mode")

The second mode of operation is the **Gateway, Direct Routing, or Direct Response mode**. In this mode the load balancer receives the incoming packet to the VIP, but doesn’t touch the destination IP of the incoming packet. Instead, it changes the destination MAC (network card) address to match one of the backends’ network card. The packet travels over the network to the backend server. The backend server also has the VIP configured, so it can accept the packet forwarded from the load balancer. The response packet from the backend is sent to the local router directly, without touching the load balancer.

It is important to note, that both the load balancer and the backends have the VIP configured. Normally, this would lead to an IP address conflict. However, we can configure the backends not to advertise the VIP on the network. This way, the only machine holding the VIP will be the load balancer as far as everyone else on the network is concerned. All incoming connections will pass through the load balancer.

The main benefit of the DR mode is the reduced load on the load balancer, as it only needs to deal with the incoming packets, but not the outgoing responses. The drawback of this mode is the more complex setup, as well as the fact that it only works on the local network.

![](/posts/ipvs-the-linux-load-balancer/ipip-01.svg "IPVS in IPIP mode")

The last mode of operation supported by IPVS is **IPIP**. This mode is similar to the DR method, but instead of changing the destination MAC address, it sends the incoming IP packet to the backend using an IP tunnel. The main benefit of this method is scalability, but cause issues when the incoming packet is too big, and the MTU limit is reached. The added overhead of the encapsulation can cause IP fragmentation. A good workaround for the MTU problem is be a larger MTU, but that is only possible when you are in full control of the network infrastructure between the load balancer and the backend.

## Our lab setup

Before we begin, we will need a lab setup. I would recommend VirtualBox as it allows for the creation of unmanaged networks you can configure to your hearts content.

![](/posts/ipvs-the-linux-load-balancer/lab-01.svg)

As a central piece of this setup we’ll install a Linux virtual machine called `router` that will replace a hardware router. Don’t worry, most routers nowadays use some Linux-variant anyway.

We will also have two networks: one for the client side, and one for our server setup. Our client will be a second virtual machine with `curl` installed, while on the server network we’ll install 3 VM’s: two for webservers and one for the load balancer.

The `client` machine will have the IP `192.168.88.2`, while the router will have the IP of `192.168.88.1` on the client network and the IP `10.0.0.1` on the server network. The load balancer will have two IP’s: `10.0.0.2` as its real IP and `10.0.1.1` as the *virtual IP* (VIP). The backends will have the IP’s `10.0.2.1` and `10.0.2.2`, respectively.

Depending on the IPVS method used the backends may or may not have the VIP configured on an internal interface.

## The NAT/Masquerade method

The NAT method is a rather traditional way of load balancing: the load balancer changes the destination IP address of any incoming packet. The setup is pretty straight forward. First, we add the IPVS rules on the load balancer:

```
# Let’s enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Let’s create the forwarding rule for the VIP (10.0.1.1)
# port 80 using the round-robin strategy.
ipvsadm -A -t 10.0.1.1:80 -s rr 

# Let’s add the backends with the "masquerade" method
ipvsadm -a -t 10.0.1.1:80 -r 10.0.2.1:80 -m
ipvsadm -a -t 10.0.1.1:80 -r 10.0.2.2:80 -m
```

Let’s try and query our VIP from the client machine:

```
# Let’s query the VIP!
$ curl http://10.0.1.1
...
```

No response. This happens because the load balancer changes the destination IP, but the backend responds directly from this changed IP. The client then drops the packet as it never knew about the backends IP address. To fix this we need to change the *default router* of the backend to the IP address of the load balancer. Routing all return packets through the load balancer will allow it to change the source IP of the return packet:

```
# Add the load balancer as the default route
ip route add default via 10.0.0.2
# Remove the original router default route
ip route del default via 10.0.0.1
```

Done! A query to the VIP now returns with the correct response:

```
# Query the VIP
$ curl http://10.0.1.1
Hello from backend 1

$ curl http://10.0.1.1
Hello from backend 2
```

As mentioned before, the NAT method has a significant drawback in terms of performance, but may be preferable for a small setup. Also note, the backend will not see the original destination IP as it is changed by the load balancer. It will, however, see the original source IP and port, which is critical for many applications.

## The Direct Routing method

As mentioned before, the Gateway / Direct Routing / Direct Response mode is a bit tricky in its operation. First, let’s set up the load balancer:

```
# Let’s create the forwarding rule for the VIP (10.0.1.1)
# port 80 using the round-robin strategy.
ipvsadm -A -t 10.0.1.1:80 -s rr 

# Let’s add the backends with the "gateway" method
ipvsadm -a -t 10.0.1.1:80 -r 10.0.2.1:80 -g
ipvsadm -a -t 10.0.1.1:80 -r 10.0.2.2:80 -g
```

Now the gateway mode is set up. We still need to the backends to have the IP address:

```
# Add the VIP to the lo interface
ip addr add 10.0.1.1/32 dev lo
```

So the VIP will be added to the local interface.

Let’s query the VIP from the client computer:

```
# Query the VIP
$ curl http://10.0.1.1
Hello from backend 1

$ curl http://10.0.1.1
Hello from backend 1

$ curl http://10.0.1.1
Hello from backend 1
```

Hmmm... as you can see the responses always come from backend 1, never from backend two... what happened?

Here’s the tricky part: backend 1 also has the VIP assigned to a network interface. When the router asks for the MAC address related to the VIP, the load balancer, as well as backend 1 and 2 respond to this query. Whichever is faster will receive the request. The router will cache the response in its ARP table.

However, this behavior can be changed. We specifically added the VIP to the `lo` (local) interface on our backends. We can now change the ARP behavior such that backend 1 and 2 don’t advertise the VIP themselves. This is done with a `sysctl` command:

```
sysctl net.ipv4.conf.eth0.arp_ignore=1
sysctl net.ipv4.conf.eth0.arp_announce=2
```

We can persist this setting by creating a file called `/etc/sysctl.d/ipvs.conf` with these settings. The only thing to be done is use `arping` from the load balancer to fix the ARP table of the router:

```
# ARPing the router (10.0.0.1) from the VIP (10.0.1.1)
arping -S 10.0.1.1 10.0.0.1
```

Now the `curl` command should yield the desired result:

```
$ curl http://10.0.1.1
Hello from backend 1

$ curl http://10.0.1.1
Hello from backend 2

$ curl http://10.0.1.1
Hello from backend 1
```

## The IPIP method

The IPIP method is very similar to the DR method in that the original IP packet is not changed. Instead, the IP packet is encapsulated into another IP packet that is sent to the target server.

As a first step, let’s set up our load balancer:

```
# Let’s create the forwarding rule for the VIP (10.0.1.1)
# port 80 using the round-robin strategy.
ipvsadm -A -t 10.0.1.1:80 -s rr

# Let’s add the backends with the "IPIP" method
ipvsadm -a -t 10.0.1.1:80 -r 10.0.2.1:80 -i
ipvsadm -a -t 10.0.1.1:80 -r 10.0.2.2:80 -i
```

Next, let’s set up our backends:

```
# Load the IPIP module
modprobe ipip

# Bring the IPIP interface up with the VIP
ip link set tunl0 up
ip addr add 10.0.1.1/32 dev tunl0
```

This *should* suffice, but if we try sending a request now, nothing will happen. This is because the incoming encapsulated packet hits the reverse-path filter in Linux. This filter checks if the packet is coming from the direction the outgoing packet would go, which is not true in this case. So, we need to turn off the `rp_filter` for the `tunl0` interface:

```
sysctl net.ipv4.conf.all.rp_filter=0
sysctl net.ipv4.conf.tunl0.rp_filter=0
```

If the backends are on the same network as the load balancer, we will also need to tune our ARP settings on the backends like with the DR method:

```
sysctl net.ipv4.conf.eth0.arp_ignore=1
sysctl net.ipv4.conf.eth0.arp_announce=2
```

Finally, let’s talk about a few pitfalls with the IPIP method. First of all, the return packet will not traverse the tunnel, it will directly go to the router of the network the backend is on. The whole point of the IPIP method is being able to place the backend on a different network than the load balancer. However, if you are not in control of that network your provider may reject the return packet as it comes from an IP address that is not in their IP range. You may think that the trivial solution would be to send the return packet back to the LB via policy routing, but the load balancer will also reject this return packet as it comes from the VIP, which is configured locally on the load balancer. (This is called a *martian* packet and is dropped immediately. Yes, this name will not age well if Elon Musk has his way.)

Another thing to talk about is IP fragmentation. Normally, connections have a maximum transmission unit (MTU) of 1500 bytes. If the incoming packet is exactly 1500 bytes in size, the additional IP encapsulation will push the forwarded packet over the MTU and will need to be fragmented, leading to performance penalties. Ideally, you can talk to your network providers to allow a larger MTU between your LB and the backends.

## Running health checks

IPVS doesn’t itself run health checks against the backends. That can be achieved by addon projects like [ldirectord](https://projects.horms.net/projects/ldirectord/). However, ldirectord only works on a small scale as it requires all VIP’s and backends to be listed in a configuration file. When building larger setups, you may want to look into writing something more elaborate.

## Building a redundant load balancer

So far we only talked about a single load balancer. What happens if this load balancer fails? Will all connections be broken when you fail over to a different machine?

Fortunately, IPVS has a mechanism to synchronize sessions across multiple LB instances. This synchronization mechanism will propagate connection information as fast as possible to a backup load balancer. You can enable it on both nodes like this:

```
# On the primary:
ipvsadm --start-daemon=master --mcast-interface=eth0

# On the secondary:
ipvsadm --start-daemon=backup --mcast-interface=eth0
```

Keep in mind, this will not propagate connections instantly, and some connections may be lost on failover. The sync daemon will also not transport the rules to the backup LB, that’s, again, something you will have to implement. 

One question is left to answer: how do you assign the VIP to one node or the other? You can, of course, use something traditional like [keepalived](https://keepalived.readthedocs.io/en/latest/introduction.html). This will serve you well in most cases.

Alternatively, you could go off the deep end and integrate your load balancer directly with your routers. I had great success doing this in the past using [Quagga](https://www.quagga.net/)/[FRRouting](https://frrouting.org/). I won’t go into details here as configuring routing protocols is way beyond this article, but it’s an interesting and surprisingly comfortable alternative to using ARP-based failover. 

## Going into production

Needless to say, the above settings are only meant as examples. There are many challenges ahead, from configuration management, to tuning and monitoring your load balancing setup. The chances are quite slim that you’ll set up IPVS yourself, but you may come in contact with it when dealing with, or debugging Kubernetes.