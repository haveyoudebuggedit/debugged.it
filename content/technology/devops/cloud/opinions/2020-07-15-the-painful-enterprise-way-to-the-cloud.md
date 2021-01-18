---
title: The (painful) Enterprise Way to the Cloud
slug: the-painful-enterprise-way-to-the-cloud
guid: https://pasztor.at/blog/the-painful-enterprise-way-to-the-cloud/
summary: Use the cloud, they said. It will be great, they said. Why is it painful then?
authors:
- janos
categories:
- Technology
images:
- posts/the-painful-enterprise-way-to-the-cloud/social.png
tags:
- Cloud
- DevOps
- Business
date: "2020-07-15T00:00:00Z"
publishDate: "2020-07-15T00:00:00Z"
mp3: podcast/The painful Enterprise Way to the Cloud.mp3
ogg: podcast/The painful Enterprise Way to the Cloud.ogg
aac: podcast/The painful Enterprise Way to the Cloud.aac
soundcloud: "https://soundcloud.com/pasztor_at/the-painful-enterprise-way-to-the-cloud"
spotify: "https://open.spotify.com/episode/52IB09ZExAg6mo1BaS6Ewy"
itunes: "https://podcasts.apple.com/podcast/the-painful-enterprise-way-to-the-cloud/id1524971396?i=1000485976063"
deezer: "https://www.deezer.com/en/episode/235907712"
google: https://podcasts.google.com/feed/aHR0cHM6Ly9wYXN6dG9yLmF0L3BvZGNhc3QueG1s/episode/cG9kY2FzdC9UaGUgcGFpbmZ1bCBFbnRlcnByaXNlIFdheSB0byB0aGUgQ2xvdWQubXAz
duration: 854
---

In recent years, my career has taken an unforeseen turn. After having worked a decade in small to medium businesses, I suddenly found myself amid very large corporations. Many of them on the move to the cloud. Some of them experiencing the move to the cloud, like a tidal wave that upends everything they have done so far in IT.

Although this post is quite snarky in its tone, there is nothing terrible or surprising about it. Large corporations are slow-moving *by design*. You can’t just simply launch a new service or move a system elsewhere. There are *rules*. These rules, like compliance, security or legal requirements, are put in place so that no individual can cause a considerable amount of damage that would tank the share price. Furthermore, to a degree, every employee should be
*replaceable* like cogs in a machine.

Let’s take a trip down the cloud lane and observe our imaginary, very traditional large corporation on their way.

## Stage 0: On-Premises

We start our trip from the on-premises setup. (Yes, it’s on-premises and not on-premise.) A typical setup includes either an own cage or a full-on datacenter. Every 3-5 years, an IT manager would have to decide what kind of hardware to purchase that will last the next 3-5 years. As we are talking about enterprises, this hardware is usually bought with a support contract, so if something breaks, the vendor is responsible for supplying the replacement hardware.

It’s a comfortable, slow way of iterating over hardware. Not much excitement. Not without problems, mind you. Sometimes systems go down. Sometimes the systems can’t handle the load. Generally, everything is a little bit slow and looks a little bit dated. 

There are *rumors* of a better world. *Rumors* that there is a way to create a fast-paced environment where deployments happen several thousand times a day, tests run automatically, and applications are fast and responsive. There are *rumors* of something called the cloud. Whoever gets there first will win the race. This *cloud* is rumored to solve
all the scaling problems, never go down and make everything just simply *better*.

## Stage 1: The Lies

This *cloud* thing has everyone in recent years a bit agitated. If you look across the big companies, it looks a little bit like the [great migration in Europe](https://en.wikipedia.org/wiki/Migration_Period). Everybody is on the move, on the move to the cloud. Every bank, telco, and other large company is announcing that they are migrating to the cloud. Job boards are filled to the brim with DevOps engineer positions to help with the migration.

If you are a manager in such a company, you are attending *cloud strategy meetings* where pretty boxes of components are drawn on PowerPoint slides. (Usually in a visually very unappealing fashion.) If you can’t quite decide which cloud you are moving to, Multi-Cloud is written all over your banners to hide the fact that there is no clear vision on what the future holds.

If you are one of the few managers who don’t yet have a *cloud strategy*, you may be subject to incredible peer pressure and [fear of missing out](https://en.wikipedia.org/wiki/Fear_of_missing_out). You can’t participate in the conversations around the proverbial water cooler.

These conversations are usually jam-packed with buzzwords and make an actual cloud engineers’ ears bleed. In Hungarian, we have a saying for this: [The blind leading the sightless](https://en.wiktionary.org/wiki/vak_vezet_vil%C3%A1gtalant).

The reasons thrown around for moving to the cloud are many: The cloud will save costs! The cloud will solve our scaling problems! The cloud will make everything faster! The cloud will never go down!

## Stage 2: Lift & Shift

At some point, it’s time to move the first workloads. Dipping their toes in to test the waters initially, just one or two projects are moved over. This move is usually done in a *lift and shift* fashion. As the name suggests, this means taking the legacy setup and moving it to the cloud with no or minimal adaptations.

Some companies chose to extend their on-premises VMware environment to the cloud. They simply use the cloud as an extension of their existing environment. Others try and do a full-on move of one of their systems. In neither scenario do they recognize that their on-premises environment had features that the cloud doesn’t.

A typical on-premises environment often has VM failover. Your traditional on-premises environment has a storage system that lets you restart a VM from a failed machine on a different machine. This storage system is usually connected over a high-performance fiber channel connection. You can treat it the same way you would treat a local disk. Since
you *own* the storage, you have the option to create even very large disks. 10 TB of high-performance storage for your Oracle database? Sure, if you have the money.

Can you do this in the cloud? Absolutely. Network-connected block storage, such as EBS from Amazon, offers you the ability to get a guaranteed throughput for your disk system. At the time of writing, you can get up to a GBit/s and up to 64.000 IOPS [on Amazon](https://aws.amazon.com/ebs/features/#Amazon_EBS_volume_types). If you max it out with a 10 TB disk, your bill will be 5000 USD per month just for the storage. That’s 60.000 USD a year. 180.000 USD over three years. 300.000 USD over five. You can reduce the price by signing a pre-commitment, but you get the picture.

You can, of course, see the problem. If you simply throw your workload on the cloud the same way you treat your on-premises setup, you will spend a *lot* of money. You see, until now, you threw oversized storage at the problem *because* you had to plan for 3-5 years. As the imaginary IT manager, you *had* to make sure you wouldn’t run into problems two years down the line. Remember, you wouldn’t get the budget to buy another shipment of metal.

The above scenario is, unfortunately, the *better* outcome. Some companies don’t even think about redundancy when moving to the cloud. They just pick an instance size that roughly fits their on-prem specs and go with that. They don’t pay attention to the fact that the instance may have a local SSD, and all data would be lost when the physical hardware dies.

You may think this is not a big problem because on-prem, everything works for years before your first disk dies. However, your on-prem disks are *new* and are also configured in a RAID. In my previous job, I ran a [Ceph](https://ceph.io/) cluster on a cloud provider. We ran on average, 10 locally connected NVMe SSDs. In two years, we had three (!) disk failures with a complete data loss.

Needless to say, by simply throwing an oversized workload into the cloud, the hoped-for scaling behavior is also not realized. Sure enough, some cloud providers offer you the ability to burst your CPU usage for a limited amount of time. However, that’s very little compared to what real horizontal scaling can do.

## Stage 3: The Shadow-ish IT

Larger organizations have little pockets of know-how by nature. There are teams in every such organization that have a higher-than-average understanding of the cloud. Maybe they run a few projects in their spare time or start using the cloud on their company projects.

Here’s the kicker: larger companies have policies. Lots of policies. These policies and procedures serve two goals: the primary goal is to protect the organization from *mistakes*. Mistakes made by people. Every action carried out is described, sometimes in excruciating detail. It has to go through multiple people until it can be done. The second
goal is to cover everyone’s behind who stuck to the rules.

You see, these *procedures* ensure that established workflows are carried out precisely the way they are supposed to be carried out. However, at this point, the cloud is not established. There are no workflows on how to *&ldquo;do cloud&rdquo;*.

You would be forgiven to think that the in-house IT operations would *drive such a change*. They could be the ones who would spearhead moving to the cloud. However, what do the in-house IT ops folks do? They are setting up and maintaining servers—lots of them. Most of the time manually. When a new server is requested, it is installed by someone by hand, and its IP address is recorded, again by hand, in an asset management tool.

This way of working does not go well with the cloud. Remember, reasons one would want to use the cloud for are cost savings, new services, and that the whole workflow can be more dynamic. Deployments can happen faster.

These are two different worlds. That’s why IT operations do not usually lead the campaign for the cloud.

&ldquo;Who then?&rdquo; &mdash; you ask.

Let’s rephrase the question: who has the *most* to gain from cloud adoption? It’s the department that has to foot the bill for the infrastructure. It’s the project owners, developers who achieve reduced costs, faster deployments, etc. But they can’t ask IT ops to move them to the cloud as they would get nothing.

That’s why the first fledgling cloud projects are usually in the realm of shadow IT. Maybe it’s a cloud account set up with the company credit card, or a Kubernetes cluster set up without IT ops knowing about it. It completely bypasses the traditional IT procedures and is also not supported.

This can be downright *dangerous* as the people experimenting with the cloud are moving customer data to a place where security is only an afterthought. Not because the cloud is unsecured, mind you, but rather the people setting it up are result-driven. Can’t access something? Let’s open up that firewall to the whole world. We’ll fix it later. Pesky object storage ACLs preventing you from accessing the files stored? `public-read`. Done.

There’s a lot missing here that enterprise compliance would require: security policies, network separation, access control, user management, you name it. Yet, these operations are tolerated.

At some point, this shadow operation grows large enough so that it can no longer be removed. In parallel, the risk is high enough such that folks start taking it seriously. Maybe the security people begrudgingly set up a VPN or proxy so that this new cloud operation can access the internal data instead of using workarounds. Maybe they offer help to start managing cloud users, integrating it with their enterprise authentication. The possibilities are endless.

One thing is common, though: the procedures are very slow to adapt.

## Stage 4: Acceptance

Slowly the cloud age dawns on our organization, and the first project owners consider actually moving to the cloud with more serious workloads. Now the procedures are somewhat established, connections are created between the on-premises network and the cloud provider, and security is appropriately managed.

The first projects to move are most likely the data lake-type projects. These teams are sitting on a massive amount of data they want to analyze. To do that, they hire data scientists responsible for taking this ridiculous data swamp and extract something useful from it.

Initially, the thought process might be that the data scientists themselves can operate an
[Apache Spark](https://spark.apache.org/), a [Kafka cluster](https://kafka.apache.org/), or maybe
[HDFS](https://hadoop.apache.org/docs/r1.2.1/hdfs_user_guide.html). However, that pipe dream soon hits a closed valve as the realization sets in that these people are good at working with data, but they are not operations specialists. They are simply not the people who will install and maintain systems on the OS-level.

Which is where the cloud comes into play. Major cloud providers and independent companies have started offering services that cater specifically to the needs of the data analysis crowd. Moving sensitive data to the cloud is rubbing a lot of traditional folks the wrong way. Still, there is simply no denying the business value of being able to better understand customers.

## Stage 5: The Future

This is not the end of the road, nor is it the end of on-premises systems. The on-premises system will exist for a long time. After a lift-and-shift project, the bill may well prompt the occasional move back to on-premises as it turns out the cloud is only cheaper if the workload makes use of the advantages.

Typically, the massive databases stay at home, and the application servers are moved to the cloud. The data transfer costs for the interconnects will add up quickly. But that won’t be visible. And, even if someone understands that the cloud may even be more expensive for specific workloads... nobody wants to be a naysayer and rock the boat.  

It’s also a fact of life that the largest cloud providers are US-owned. This can present a problem for some European customers, such as when government projects are on the line. Which is why the [GAIA-X](https://www.data-infrastructure.eu/GAIAX/Navigation/EN/Home/home.html) project even exists.

Does this mean no European company is moving to the cloud? Of course not. They don’t have a choice. Cloud providers offer a vast array of services the in-house IT doesn’t have a snowball’s chance in hell to catch up with. Even banks are threatened by fintech startups and are *forced* to move faster. More user-friendly interfaces, faster development cycles. Gone is the age of terrible user interfaces that only work with Java applets in Internet Explorer.

Let me reiterate: there is nothing wrong with this. Big enterprises are built to last. That comes with a certain rigidness. They are slow to accept change *by design*. But, you don’t see large enterprises going under all that often. This is more than you can say about startups. So make fun as much as you want, there’s a method to the madness. 

I, for one, have my stash of popcorn ready. [IT monocultures](/blog/monocultures) are becoming
a real problem. A potential shift in geopolitics could yield some fun to watch migration projects. (Fun for the bystander, not so much the person who has to do them.)

Fun times, people. Fun times.