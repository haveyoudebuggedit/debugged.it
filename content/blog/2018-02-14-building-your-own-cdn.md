---
slug: building-your-own-cdn
guid: https://pasztor.at/blog/building-your-own-cdn/
authors: 
- janos
categories:
- Technology
date: "2018-02-14T00:00:00Z"
publishDate: "2018-02-14T00:00:00Z"
summary: Fresh from the hold-my-beer department, why don't we build our own little CDN? Oh, and it actually makes sense.
images:
- posts/building-your-own-cdn.png
preview: posts/building-your-own-cdn.jpg
tags:
- Containers
- DevOps
- Cloud
title: Building your own CDN for Fun and Profit
---


As you can (hopefully) see from this site, I like my pages *fast*. Very, very fast. Now, before we jump into this, let
me be very clear about it: using a CDN will only get you so far. If your site is slow because of shoddy frontend work,
a CDN isn't going to help you much. You need to get your frontend work right first. However, once you've optimized
everything you could, it's time to look at content delivery.

{{% info %}}
**Update one year later:** the CDN is now disabled. If you are curious about the reasons, scroll to the end.
{{% /info %}}

My main problem was that even though you could get the inital website load with a single HTTP request, my server being
hosted in Frankfurt, the folks from Australia still had to wait up to 2-3 seconds to get it. Round trip times of over
300 ms and a lot of providers inbetween made the page load just like any other Wordpress site.

So what can we do about it? One solution, of course, would be the use of a traditional CDN. However, most commercial CDNs pull the data from your server on request and then cache it for a while.

![A user requests a page from the CDN, which takes 50 ms. The CDN then requests it from the Origin, which takes 300 ms. Without a CDN the user requests directly from the Origin.](posts/building-your-own-cdn/cdn.svg)

Using a traditional CDN the initial page load is slower than without it, since the CDN is a slight detour for the
content. This is not a problem if you have a high traffic site since the content stays in the cache all the time. If, on the other hand, you are running a small blog like I do the content drops out of the cache pretty much all the time. So, in effect, **a traditional pull-CDN would make this site slower**. I could, of course, use a push-CDN where I can upload the content directly, but those seem to be quite pricey in comparison to what I'm about to build.

## How do CDNs work?

Our plan is clear: on our path to world domination we need to make our content available everywhere *fast*. That means
our content needs to be close to the audience. Conveniently, there are a lot of cloud providers that offer cheap
virtual servers in multiple regions. We can just put our content on, say, 6 servers and we're good, right?

Well, not so fast. How is the user going to be routed to the right server? Let's take a look at the process of actually
getting a site. First, the users browser uses the Domain Name System (DNS) to look up the IP address of the website.
Once it has the IP address, it can connect the website and download the requested page.

![The user first looks up the IP of pasztor.at from the DNS service. Then the user requests the page from the IP looked up.](posts/building-your-own-cdn/dns.svg)

If we think about it on a high level, the solution is quite simple: we need a smart DNS server that does a GeoIP
lookup on the requesting IP address and returns the IP address closest to it. And indeed, that's (almost) how commercial CDNs do it. There is a bit more engineering involved, like measuring latencies, but this is basically how it's done. 

### Making the DNS servers fast

Now the next question arises: how do we make the DNS server fast? Getting the website download to go to the closest node
is only half the job, if the DNS lookup has to go all the way around the planet, that's still a HUGE lag.

As it turns out, the infrastructure underpinning the internet is uniquely suitable to solve this problem. Network
providers use the *Border Gateway Protocol* to tell each other which networks they can reach and *how many hops* away
they are. The end user ISP then, in most cases, takes the shortest route to reach the destination.

If we now advertise the IP addresses in multiple locations, the DNS request will always be routed to the closest node.
This is called BGP Anycast.

### Why not use BGP Anycast for the website download?

Wait, hold on, if we can do this, why don't we simply use BGP to route the web traffic? Well, there are three reasons.

First of all, doing BGP Anycast requires control over the network hardware and a pool of at least 256 IP addresses,
which is way over our budget.

Second, BGP routes are not *that* stable. While DNS requests only require a single packet to be sent in both directions,
HTTP (web) requests require establishing a connection to download the content. If the route changes or the connection is
unstable, the HTTP connection could broken. That adds a lot of complexity for a project of this scale.

And finally, the lowest count of hops, which is the basis of BGP route calculations, does not guarantee the lowest round
trip time. A hop across the ocean may be just one hop, but it's a damn long one.

{{% tip %}}
**Further reading:** LinkedIn Engineering has a
[wonderful blog post about this topic](https://engineering.linkedin.com/network-performance/tcp-over-ip-anycast-pipe-dream-or-reality).
{{% /tip %}}

## Setting up DNS

Since we have established that we can't run our own BGP Anycast, this means we can also not run our own DNS servers. So
let's go shopping! ... OK, as it turns out, DNS providers that offer BGP Anycast servers and latency-based routing
are a little hard to come by. During my search I found only two, the rather pricey [Dyn](https://dyn.com/) and the
dirt-cheap [Amazon Route53](https://aws.amazon.com/route53/). (Update: as it turns out, [DNS Made Easy](https://dnsmadeeasy.com/)
also does latency-based routing.)

Since we are cheap, Route53 it is. We add our domain and then start setting up the IPs for our machines. We need as many
DNS records as we have servers around the globe (edge locations), and each record should look like this:

![Route53 latency-based routing should be set up in Route53 by creating A records with the IP of the edge location, and then setting the routing policy to &quot;latency&quot;. The set ID should be something unique, and the location should be the one closest to our edge location.](posts/latency-based-routing.png)

{{% tip %}}
**Tip**: it is useful to set up a health check for each of the edge locations so they are removed if they go down.
{{% /tip %}}

## Distributing content

The next issue we need to tackle is distributing content. Each of your edge nodes needs to have the same content. If you
are using a static site generator like [Jekyll](https://jekyllrb.com/), your task is easy: simply copy the generated 
HTML files on all servers. Something as simple as rsync might just do the trick.

If you want to use a content editing system like Wordpress, you have a significantly harder job since it is not built to
run on a CDN. It [can be done](/blog/intercontinental-docker-swarm), but it's not without its drawbacks, and the
distribution of static content is still a problem. You may have to create a distributed object storage for that to fully
work.

## Using SSL/TLS certificates

The next pain point is using SSL/TLS certificates. Actually, let's call them what they are: x509 certificates. Each of
your edge locations needs to have a valid certificate for your domain. The simple solution, of course, is to use
[LetsEncrypt](https://letsencrypt.org/) to generate a different certificate for each, but you have to be careful. LE has
a rate limit, which I ran into on one of my edge nodes. In fact, I had to take the London node down for the time being
until the weekly limit expires.

However, I am using [Traefik](https://traefik.io/) as my proxy of choice, which supports using a distributed key-value
store or even [Apache Zookeeper](https://zookeeper.apache.org/) as the backend for synchronization. While this requires
a bit more engineering, it is probably a lot more stable in the long run.

## The results

Time for the truth, how does my CDN perform? Using [this tool](https://latency.apex.sh/?url=https%3A%2F%2Fpasztor.at&compare),
let's see some global stats:

![Oregon: 246ms, California: 298ms, Ohio: 227ms, Virginia: 108ms, Ireland: 217ms, Frankfurt: 44ms, London: 110ms, Mumbai: 870ms, Singapore: 517ms, Seoul: 253ms, Tokyo: 150ms, Sidney: 358ms, Sao Paulo: 911ms](posts/latency.png)

As you can see, the results are pretty decent. I might need two more nodes, one in Asia and one in South America to get
better load times there.

{{% info %}}
**Update:** After I've made it to the [Hacker News](https://news.ycombinator.com/news) front page (wow!), I had a chance
to collect a bit of real usage data using Google Analytics:

![](posts/real-world-cdn-usage.png)

Bottom line: I really need that Singapore node. The load times in India are above the desired 1 second.
{{% /info %}}

## Frequently asked questions

When I do projects like this, people usually ask me: *"Why do you do this? You must like pain."* Yes, to some extent I
like doing things differently just for the sake of exploring new options and technologies, building your own CDN may
make a lot of sense. Let's address some of the questions about this setup.

Let's be clear: if a commercial provider comes out with an affordable push CDN that allows me to do nice URLs, SSL and
custom headers, I'll absolutely throw money at them and stop running my own infrastructure. As fun as it was to build,
I have enough servers to run without this. 

### Why don't you just use CloudFlare?

CloudFlare is a wonderful tool for many, but as outlined above, CDNs drop unused content from their cache. On other
sites that I'm managing I see a cache rate of about 75% with the correct setup. Having your own CDN means 100% of the
content is always in cache, and there are no additional round trips to the origin server.

### Why don't you use S3 and CloudFront?

Amazon S3 has an option to host static websites, and it works in conjunction with CloudFront. However, it does not allow
you to set custom headers for caching, nice URLs, etc. For that, you need Lambda@Edge, a tool that lets you run code
on the CloudFront edge nodes. Lambda@Edge, however, has the same problem as CDNs: if it doesn't receive requests for 
a certain time, the container running it is shut down and needs up to a second to boot up.

### Why don't you use Google AMP?

Google AMP only brings benefits when people visit your site from the Google search engine. Most of my traffic does
not come from Google so that won't solve the problem. So it really only benefits Google, nobody else. Oh, and I'm
perfectly capable of building a fast website without the dumbed down HTML they offer.

### Who cares? 3 seconds is a wonderful load time!

I'm a DevOps engineer who specializes in delivering content. If anyone, I should have a website that's fast around the
globe, no?

Oh, and I like to flip Google AMP off because it's a terrible technology. Not that they'd care.

## Build your own

Now it's up to you: do you want to build your own CDN?
[The source code for mine is right there on my GitHub.](https://github.com/janoszen/pasztor.at/tree/legacy/_ansible)
Go nuts!  

## One year later...

One year later I have decided to tear down my CDN. The reasons boil down to the fact that I already mentioned: this 
system is not production ready. A couple of months ago some of my regular readers pointed out that my page sometimes
comes back with a 404. Upon closer investigation it turned out that the reason for this 404 was the CPU usage.

Not the CPU usage of my machines but of my &ldquo;neighbors&rdquo;. Logging into my server I saw a CPU steal time of up
to 90%:

![](posts/steal-time.png)

This high CPU usage caused Traefik to drop the backend due to a timeout, which in turn caused the 404 error. As I
installed Uptime Robot to check my site I realized that this is quite a common occurrence and happened 3-4 times a week.

One may have misgivings about Amazon Lightsail (the virtualization platform under the CDN), but it point to a larger
picture: with a distributed system like this monitoring and quality assurance is crucial. My conversations with other
engineers running globally distributed systems confirmed this: there is absolutely no magic in building a CDN like
this. Running it, however, realing with the underlying platform performance, or network performance, is the hard part
that requires constant effort and maintenance.

As a result I have rebuilt my setup on a single server, with an
[immutable infrastructure](https://pasztor.at/blog/immutable-infrastructure) and proper monitoring, with Terraform. If
you want to know more about [how I did it, read the article here](/blog/immutable-infra-1).
