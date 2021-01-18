---
categories:
- Technology
slug: forwarding-ip-traffic-through-a-vpn
guid: https://pasztor.at/blog/forwarding-ip-traffic-through-a-vpn/
date: "2019-12-18T00:00:00Z"
publishDate: "2019-12-18T00:00:00Z"
summary: 'You have two servers and want to serve the traffic for an IP address on one server from another server? '
images:
- posts/forward-ip-traffic-vpn.png
preview: posts/forward-ip-traffic-vpn.jpg
tags:
- DevOps
- Networks
title: Forwarding IP traffic through a VPN
authors:
- janos
---

One of my readers has posed me the question a while ago: I have two servers, A and B. How do I run a webserver on 
server B, but still forward the traffic from server A's IP address to it?

## The test setup

Before we start our little experiment, let's make one thing abundantly clear: creating a firewall for a production-grade server is way, way out of scope for this document, so we will use Docker containers to simulate our environment.

First of all we will create a network:

```bash
docker network create \
    --gateway 192.168.0.254 \
    --subnet 192.168.0.1/24 \
    test
```

Then create the first (A) server:

```bash
docker run \
    --cap-add NET_RAW \
    --cap-add NET_ADMIN \
    --name server-a \
    --network test \
    --ip 192.168.0.1 \
    -dti \
    ubuntu
```

Then we create the second (B) server with nginx in it:

```bash
docker run \
    --cap-add NET_RAW \
    --cap-add NET_ADMIN \
    --name server-b \
    --network test \
    --ip 192.168.0.2 \
    -d \
    nginx
```

If we did everything correctly, `docker ps` should yield the following:

```bash
CONTAINER ID  IMAGE  COMMAND  CREATED        STATUS   PORTS   NAMES
1d8d2a56af5b  nginx  "ngin…"  3 seconds ago  Up 2 seconds  80/tcp  server-b
1566ff29bf76  ubuntu "/bin…"  14 seconds ago Up 13 seconds          server-a
```

Next, let's install some utilities:

```bash
$ docker exec -ti server-a /bin/sh -c "apt update && apt install -y tcpdump iproute2 iptables"

$ docker exec -ti server-b /bin/sh -c "apt update && apt install -y tcpdump iproute2 iptables"
```

Let's check if our servers have the correct IP addresses:

```bash
$ docker exec -ti server-a ip addr show dev eth0
36: eth0@if37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:c0:a8:00:01 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.0.1/24 brd 192.168.0.255 scope global eth0
       valid_lft forever preferred_lft forever

$ docker exec -ti server-b ip addr show dev eth0
38: eth0@if39: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:c0:a8:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.0.2/24 brd 192.168.0.255 scope global eth0
       valid_lft forever preferred_lft forever
```

If your IP's are different at this point, note them and adjust them accordingly. As a final test, let's open 
http://192.168.0.2 from the host machine:

```bash
$ curl http://192.168.0.2
```

The result should be the source code of the default nginx page. Now, our goal is to be able to access the webserver
running in `server-b` on the IP address `http://192.168.0.1`

## The simple answer

Let's look at it from the perspective of `server-a`. You can run a `tcpdump` to see the packets coming in when a 
connection is attempted:

```bash
$ docker exec \
    -ti \
    server-a \
    tcpdump \
        -i eth0 \
        -n \
        port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
22:15:17.405131 IP 192.168.0.254.56988 > 192.168.0.1.80: Flags [S], seq 1891979943, win 29200, options [mss 1460,sackOK,TS val 1916994860 ecr 0,nop,wscale 7], length 0
22:15:17.405150 IP 192.168.0.1.80 > 192.168.0.254.56988: Flags [R.], seq 0, ack 1891979944, win 0, length 0
```

Pay attention to the IP's. The first packet is coming in, then a response is going out. If you look at the flags, the
first one is a SYN (to open a connection), the response is a RST (to reject the connection). What we want to do is get
server A to, instead of rejecting the packet, forwarding it to `server-b`. In order to do that we need to *hack*
server A to change the destination IP address of this packet:

```bash
$ docker exec -ti server-a iptables -t nat -A PREROUTING -d 192.168.0.1 -p tcp --dport 80 -j DNAT --to 192.168.0.2
```

After this let's run the test again and try to query `http://192.168.0.1`, this is the result:

```bash
$ docker exec -ti server-a tcpdump -i eth0 -n port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:18:24.217281 IP 192.168.0.254.57022 > 192.168.0.1.80: Flags [S], seq 2407047481, win 29200, options [mss 1460,sackOK,TS val 1917181674 ecr 0,nop,wscale 7], length 0
22:18:24.217348 IP 192.168.0.254.57022 > 192.168.0.2.80: Flags [S], seq 2407047481, win 29200, options [mss 1460,sackOK,TS val 1917181674 ecr 0,nop,wscale 7], length 0
22:18:25.242603 IP 192.168.0.254.57022 > 192.168.0.1.80: Flags [S], seq 2407047481, win 29200, options [mss 1460,sackOK,TS val 1917182699 ecr 0,nop,wscale 7], length 0
22:18:25.242666 IP 192.168.0.254.57022 > 192.168.0.2.80: Flags [S], seq 2407047481, win 29200, options [mss 1460,sackOK,TS val 1917182699 ecr 0,nop,wscale 7], length 0
```

So you can see, the packet with the destination IP `192.168.0.1` is coming in, and the packet to `192.168.0.2` is going
out. Please note that the source IP address has *not changed*. This is called a destination NAT. In contrast, changing
the source IP is called a source NAT. Destination and source NAT together give a full NAT.

Now, this is all cool, but it still isn't working, so let's take a look what's happening in the mean time on server B:

```bash
$ docker exec \
    -ti \
    server-b \
    tcpdump \
        -i eth0 \
        -n \
        port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:20:56.454868 IP 192.168.0.254.57046 > 192.168.0.2.80: Flags [S], seq 3125382617, win 29200, options [mss 1460,sackOK,TS val 1917333912 ecr 0,nop,wscale 7], length 0
22:20:56.454884 IP 192.168.0.2.80 > 192.168.0.254.57046: Flags [S.], seq 498628625, ack 3125382618, win 28960, options [mss 1460,sackOK,TS val 3129097399 ecr 1917333912,nop,wscale 7], length 0
22:20:56.454911 IP 192.168.0.254.57046 > 192.168.0.2.80: Flags [R], seq 3125382618, win 0, length 0
```

Something very strange is going on. We receive the incoming packet with a SYN flag, to which our server correctly 
responds with a SYN-ACK to establish a connection, but then the client sends an RST, rejecting the SYN-ACK packet sent
in response.

This is happening because the host machine that originally sent the packet expect the answer from `192.168.0.1`, and
not `192.168.0.2`, so it rejected the packet. In a production network environment you may also see that this packet
simply disappear, but more on that later.

So how do we change the path of the returning packet? We can't intercept it on server A because our return packet
is not going through server A, it is directly going back to the querying host.

In order to make the response packet go back to server A we can ask server A to not only do a DNAT, but also an SNAT, 
so it will appear to server B that server A was actually opening the connection:

```bash
$ docker exec \
    -ti server-a \
    iptables \
        -t nat \
        -A POSTROUTING \
        -d 192.168.0.2 \
        -p tcp \
        --dport 80 \
        -j SNAT \
        --to 192.168.0.1
```

Hooray! After this everything will work! However, if we inspect the traffic on server B we will see a strange thing:

```bash
$ docker exec \
    -ti \
    server-b \ 
    tcpdump \
        -i eth0 \
        -n \
        port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:26:16.070868 IP 192.168.0.1.57130 > 192.168.0.2.80: Flags [S], seq 3230417142, win 29200, options [mss 1460,sackOK,TS val 1917653529 ecr 0,nop,wscale 7], length 0
```

The traffic *appears* to be coming from `192.168.0.1` and the information where the connection came from is completely
lost.

## Routing to the rescue

So let's remove that last iptables rule:

```bash
$ docker exec \
    -ti \
    server-a \
    iptables \
        -t nat \
        -D POSTROUTING \
        -d 192.168.0.2 \
        -p tcp \
        --dport 80 \
        -j SNAT \
        --to 192.168.0.1
```

Instead, we should tell server B to send the traffic back via server A. This can be done using a simple routing rule:

```bash
$ docker exec \
    -ti \
    server-b \
    ip route del \
        default \
        via 192.168.0.254 \
        dev eth0
$ docker exec \
    -ti \
    server-b \
    ip route del \
        192.168.0.0/24 \
        dev eth0 \
        proto kernel \
        scope link \
        src 192.168.0.2
$ docker exec \
    -ti \
    server-b \
    ip route add \
        192.168.0.1/32 \
        dev eth0 \
        proto kernel \
        scope link \
        src 192.168.0.2
$ docker exec \
    -ti \
    server-b \
    ip route add \
        default \
        via 192.168.0.1
```

This way the routing will be reconfigured on server B such that it can only send packets directly to server A, and
everything else goes over server A. This way server A acts as a router for server B and translates the packets back to
the IP address of server A.

However, if you observe closely, you will not have internet access on server B now. So let's fix that. First of all, 
let's find out what's going on. First, let's open a `tcpdump` on server B, but this time around let's use the `-e` flag
to show us the mac addresses too:

```bash
$ docker exec \
    -ti \
    server-b \
    tcpdump -i eth0 -n -e
```

Then, while leaving the terminal open, on a second terminal let's run an update:

```bash
$ docker exec \
    -ti \
    server-b \
    apt update
```

In the first terminal we will see the outgoing packets, and, if you observe closely, the destination MAC address
of server A:

```bash
$ docker exec \
    -ti \
    server-b \
    tcpdump \
        -i eth0 \
        -n \
        -e
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
08:17:05.040445 02:42:c0:a8:00:02 > 02:42:c0:a8:00:01, ethertype IPv4 (0x0800), length 74: 192.168.0.2.34374 > 212.211.132.250.80: Flags [S], seq 3602023972, win 29200, options [mss 1460,sackOK,TS val 2680231590 ecr 0,nop,wscale 7], length 0
```

You will also see that there are no return packets, nothing. It seems as if server A would just simply swallow the 
packets and do nothing with them. We can check this when looking at the whole thing from server A and running the
update again. Let's take a look:

```bash
$ docker exec \
    -ti \
    server-a \
    tcpdump \
        -i eth0 \
        -n \
        -e
[...]
08:21:43.772877 02:42:c0:a8:00:01 > 02:42:c0:a8:00:02, ethertype IPv4 (0x0800), length 102: 192.168.0.1 > 192.168.0.2: ICMP redirect 151.101.14.133 to host 192.168.0.254, length 68
```

As you can see, the response is an ICMP redirect. This is supposed to indicate to server B that it should send the 
packet directly to the router, but since server B has a different network configuration, this won't work. So let's
disable ICMP redirects. Since we are running in a Dockerized environment, we will need to do this on the host machine:

```bash
$ sudo sysctl net.ipv4.conf.all.send_redirects=0
```

This, however, will not result in a working setup. The incoming packets will still be swallowed.

## Building a VPN

Before things get too complicated, let's move this setup a bit into the real world. Normally, in a real world setup, you won't be able to manipulate packets this way. Normally hosting providers filter packages going out with the wrong IP address.

{{% warning %}}
**Warning!** This section demonstrates a very simple way to build a VPN with OpenVPN, but this is not production grade. Please look up how to configure OpenVPN properly!
{{% /warning %}}

So in order to actually make this work, we will need a private connection between the two servers. To simulate this, let's remove and re-create the two containers:

```bash
$ docker rm -f server-a server-b
$ docker run \
    --cap-add NET_RAW \
    --cap-add NET_ADMIN \
    --device /dev/net/tun:/dev/net/tun \
    --name server-a \
    --network test \
    --ip 192.168.0.1 \ 
    -dti \
    ubuntu
$ docker run \ 
    --cap-add NET_RAW \
    --cap-add NET_ADMIN \
    --device /dev/net/tun:/dev/net/tun \
    --name server-b \
    --network test \
    --ip 192.168.0.2 \
    -d \
    nginx
$ docker exec \
    -ti \
    server-a \
    /bin/sh \
        -c \
        "apt update && apt install -y tcpdump iproute2 iptables openvpn"
$ docker exec \
    -ti \
    server-b \
    /bin/sh \
        -c \
        "apt update && apt install -y tcpdump iproute2 iptables openvpn"
```

Now that we have our two servers ready, let's create the OpenVPN server on server A:

```bash
$ cat <<EOF | docker exec -i server-a /bin/sh -c "cat >/etc/openvpn/openvpn.conf"
dev tun
ifconfig 10.8.0.1 10.8.0.2
secret /etc/openvpn/static.key
EOF
```

Let's also create the VPN client on server B:

```bash
$ cat <<EOF | docker exec -i server-b /bin/sh -c "cat >/etc/openvpn/openvpn.conf"
remote 192.168.0.1
dev tun
ifconfig 10.8.0.2 10.8.0.1
secret /etc/openvpn/static.key
EOF
```

Now we can create the secret key that will be used to authenticate the tunnel:

```bash
$ docker exec \
    -ti \
    server-a \
    openvpn \
        --genkey \
        --secret /etc/openvpn/static.key
$ docker cp \
    server-a:/etc/openvpn/static.key \
    static.key
$ docker cp \
    static.key \
    server-b:/etc/openvpn/static.key
```

We can now start the VPN on the two ends:

```bash
$ docker exec \
    -dti \
    server-a \
    openvpn /etc/openvpn/openvpn.conf
$ docker exec \
    -dti \
    server-b \
    openvpn /etc/openvpn/openvpn.conf
```

This way we now have a private point to point connection between the two servers. Our goal is now to NAT port 80
to the private address of server B (`10.8.0.2`) and any outgoing packets from server B from this address should also
go via the tunnel instead of going to the default gateway.

First the rules for server A:

```bash
$ docker exec \
    -ti \
    server-a \
    iptables \
        -t nat \
        -A PREROUTING \
        -d 192.168.0.1 \
        -p tcp \
        --dport 80 \
        -j DNAT \
        --to 10.8.0.2
```

Now comes the tricky part, server B. Normally all packets are routed according to the default routing table:

```bash
$ docker exec -ti server-b ip route
default via 192.168.0.254 dev eth0 
10.8.0.1 dev tun0 proto kernel scope link src 10.8.0.2 
192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.2
```

Now we will add a second routing table on server B, specifically instructing it to send all traffic via the VPN to 
server A:

```bash
$ docker exec -ti server-b \
    /bin/sh -c "echo \"1   test\" >> /etc/iproute2/rt_tables"

$ docker exec -ti server-b \
    ip rule add from 10.8.0.2 table test

$ docker exec -ti server-b \
    ip route add default via 10.8.0.1 dev tun0 table test

$ ip route flush cache
```

That's it! Now you should see your webserver on `http://192.168.0.1` and the source address should be preserved!
