---
slug: monocultures
guid: https://pasztor.at/blog/monocultures/
authors:
- janos
categories:
- Technology
date: "2019-06-12T00:00:00Z"
publishDate: "2019-06-12T00:00:00Z"
summary: Monocultures already caused the death of one strand of banana. What can we learn from it in IT?
images:
- posts/monocultures.png
preview: posts/monocultures.jpg
tags:
- DevOps
- Software Development
title: The danger of monocultures
---

In the 1950's a bananas looked much different than today. They were straight instead of curved, and they didn't have the signature bright yellow color todays bananas have. That's because in the 50's most of the worlds bananas were from the [Gros Michel or Big Mike](https://en.wikipedia.org/wiki/Gros_Michel_banana) variant. By the 60's, however, this kind of banana was all but gone. It was attacked by a particular kind of [fungus](https://youtu.be/9H0dy8fv33M) called the Panama disease that this variant was susceptible to and was replaced by the Cavendish, the banana we know and love (or hate) today.

The demise of the Big Mike was brought about by it being a *monoculture*. Both the Big Mike and the today 
popular Cavendish are banana types that are grown using a method [parthenogenesis](https://en.wikipedia.org/wiki/Parthenogenesis). This means that the banans are *essentially clones* of each other, there is no genetic difference between them, no possibility to have a mutation that would protect them from
a certain kind if disease.

But what does this have to do with IT or DevOps in particular?

Think about a larger SaaS (System as a Service) solution where customers sign up and use your service using a web interface. Each of your customers uses the service on a daily basis and they store some data on the platform.

One day a customer has the absolutely brilliant idea that they will write a little program that scrapes their data off your platform. Instead of doing the sensible thing and ask you for the data, they start hitting your website with thousands and thousands of request.

You may, of course, have implemented some sort of an application firewall, which may mitigate this issue. But imagine if this behavior is very close to the normal use case, and the firewall can't differentiate between legitimate requests and illegitimate ones. Furthermore, what if not just one customer does this. Or, to make matters worse, what if you didn't prepare for the particular kind of issue the customer is causing.

In this case the whole system may just go down. Not just for that one customer, for everyone. That's because your system was a *monoculture*. It was a single, tightly coupled system and the failure of one component took the whole system down.

## Partitioning

Now imagine if you partition the customers into chunks. Customers A-F go on server 1, G-N go on server 2, and so on. In this case an outage will (hopefully) only affect a part of your customers. An outage is still bad, but at least it will only affect a part of your customers.

You will, of course, need to make sure that there are no dependencies between your partitions, otherwise all your hard work will be for not. If you, for example, use a shared filesystem like CephFS under all your machines, a failure or slowdown in CephFS will cause a system-wide problem. *You put, as it were, all your proverbial eggs in your proverbial basket.*

Let's look at a different example. Say, you run a webhosting provider. You may have engineered the superduper system where all your customers live on a distributed system. It's the hottest new tech, where customers can transparently be moved from one server another. But again, there are components that can fail and cause a general outage. Common databases, load balancers, etc.

Note that these are *not* single points of failure since they are redundant. They are single components of failure.

So what can you do? First of all, if you are into engineering big systems, stop thinking about a single system. Put some eggs into different baskets. Make sure that when something goes bad (and in a system large enough it will), only a part of your customers will be affected.

The easiest way to do this is what Slack, ZenDesk and others do: give each customer a subdomain, say
`customername.example.com` and point each customer to the server or cluster they have been assigned to. DNS APIs are commonplace these days, and the DNS system is pretty robust so it is unlikely to fail.

You will, of course, have to implement migration procedures that move customers, especially if you are using single servers. However, since these migrations only affect single customers, a small downtime for one customer may be acceptable, so you don't have to over-complicate the whole issue. 

Alternatively you could engineer your system in such a way that in case of a migration the application writes data into both systems. This can even be done in an atomic fashion using [two-phase commits](https://en.wikipedia.org/wiki/Two-phase_commit_protocol).

## Third parties

Let's say you have done everything in your power to avoid monocultures in your own architecture and you are reasonably certain that the most likely events will not cause a system-wide outage. This will, of course, not rule out everything that could cause a system-wide outage. In the above example an outage of the DNS servers could, for instance, cause such a problem, but it is extremely unlikely.

However, if you look at a web application, apart from your own resources a lot of web applications include third party JavaScript needed to run the application. When the cloud provider that hosts this third party JavaScript has an issue, your site may also be affected, even though you don't even host anything on that cloud provider.

This happened to me in 2017, when [AWS had a major problem with their S3 service](https://www.vox.com/2017/3/2/14792636/amazon-aws-internet-outage-cause-human-error-incorrect-command) in the us-east-1 region. I was investigating a purported outage on a site that seemingly had nothing to do with S3, but was still hanging in the page load process. It turned out that the site in question loaded a third party JavaScript from an S3 bucket, which caused it to hang. 

And this is not the only time this happened. [Google](https://www.theverge.com/2019/6/2/18649635/youtube-snapchat-down-outage) recently had a massive outage too. No provider is immune to that.

The more of these third party dependencies you have for your service, the more likely it is that a problem in any of these providers will affect you.

On a semi-political side note, that's why I think we need competition in the cloud market. AWS CloudFront and CloudFlare have an [over 60% market share](https://www.datanyze.com/market-share/cdn/cloudflare-cdn-market-share) in the CDN market among themselves and CloudFlare alone handles [5-10% of the global internet traffic](https://www.wired.com/story/cloudflare-spectrum-iot-protection/). This is worryingly high because the danger that an outage of these systems is going to affect basically and website through embedded JavaScript or the unavailability of a required third party API is real. This problem will become more pronounced as even big banks are feeling the pressure to move into the cloud.

I think we need to learn from what happened to nature and avoid monocultures. They are prone to diseases, just as our IT monocultures are susceptible to outages and attacks.

Bananas!
