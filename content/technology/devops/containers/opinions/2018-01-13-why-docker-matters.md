---
slug: why-docker-matters
guid: https://pasztor.at/blog/why-docker-matters/
authors:
- janos
categories:
- Technology
date: "2018-01-13T00:00:00Z"
publishDate: "2018-01-13T00:00:00Z"
summary: Have you ever wondered what all the fuss is about with this Docker thing?
  Are you having a hard time convincing your colleagues to take it seriously? Well
  then, read on, I'm going to lay it all out for you.
images:
- posts/why-docker-matters-and-why-you-should-care.png
preview: posts/why-docker-matters-and-why-you-should-care.jpg
tags:
- Containers
- DevOps
title: Why Docker matters and why you should care
---

## The Good Old Days

Let's go back in time, oh, ten-fifteen years. You see a sysadmin (the webmaster had died by then) installing a server by
typing commands into a console. By hand. That's right, no automation. Maybe a few have caught on to configuration
management by then, but a vast majority of people pretty much installed their servers by hand. Or made server images,
which came with its own set of problems.

This, of course, led to a system where every server was somewhat unique. Larger companies had wikis where they'd
document commands to copy-paste in order to carry out certain tasks like installing a server, or even something as
simple as running a kernel update.

Updates were rarely applied, and even if they were, absolutely not on time. If a kernel update had to be applied even
to a rack full of servers (~20-30) that took an experienced sysadmin anywhere between a day and a week. Just keeping
hundreds of servers running was a full time job for more than a dozen people.

Yeah, it was pretty much the wild west in terms of server security, but nobody seemed to pay much attention to it.

## Presenting: Configuration Management

In hindsight the problem with this system is easy to see. Infrequent security updates, inconsistent servers, not to
mention that testing a configuration involved building a *complete copy of the system* (by hand, of course).

Now, some smart folks realized that this was a problem and the configuration management software was born. The very 
first (apart from a few hand-written shell scripts) was perhaps Cfengine, followed by Puppet, Chef, etc.

It didn't help adoption that the first versions of these were... *not very good*, and I'm being polite. At first they
were buggy, complicated to set up, and generally more hassle than they were worth until you ran at least a rack full
of servers. However, when set up, they became incredibly powerful tools. I remember a friend saying: <q>Ops needs a
room full of dudes to run a couple of racks and I'm managing 70 machines on my own.</q>

Some CM tools, like Puppet, invented their own language (DSL) and followed a *declarative* style of configuration. This
meant that in the configuration you would describe the state of the server you'd want it to be in, and Puppet would
do its best to reach that state. Needless to say that proved tricky at times.

Other CM software opted to use an already existing language (Ruby, Python) and follow an *imperative* model. This meant
that instructions were executed instead of describing the state of the server.

Sysadmins had to learn programming, essentially, which is a trend that we continue to observe as things develop. That's
why it was originally called DevOps, before the marketing and sales people subverted the term to cover everything under
the sun.

However, no matter what CM tool you chose, among many smaller issues, there was still one giant problem: in order to 
fully test a configuration, you'd need to set up a server at least reasonably similar to your production environment.
You could, of course, write small-scale tests for your individual modules, but more often than not sysadmins would
simply SSH into a production server and edit the configuration "Yolo!" style. Some would then be disciplied enough
to apply the same change to Puppet, but in stressful situations many would forego that step and simply leave the CM
tool turned off. Leaving it turned off meant that the infrastructure slowly eroded over time.

## Docker & Co take over

While the sysadmin world was warming up to the idea of configuration management, a different technology was evolving.
Starting with OpenVZ, then transitioning on to LXC, the Linux kernel was slowly getting support for running containers.

Containers are different to virtual machines in that they don't have their own system kernel. (The thing that deals with
hardware and switching between processes.) This means that the overhead and cost of running a virtual machine was very
minimal. Initially this was used to simply run low-cost virtual machines, but with Docker everything changed.

Docker built in top of the containerization tools built into the Linux kernel. However, instead of running a permanent
virtual machine that would need configuration management and maintenance, it was built to run a prepared virtual machine
image, and every update should be done by simply stopping the old container and launching a new container with the
updated image.

Furthermore, Docker also gives us a new and very simple tool for actually building that image file. The `Dockerfile`
is as close to a shell script as it gets, and not only builds the image, but also *documents the building process*,
essentially producing a (more or less) reproducible build.

## Why this is a change

Admittedly, updating servers is hard. Every time you update a live server you run the risk of having more downtime
than expected. What's worse the more updates you postpone the more updates you have to run when you actually do them,
making the risk of something happening even greater. If you wanted to test the updates beforehand, you'd have to run a
separate system that is an exact copy of your production system. While this may not seem to be a problem, anyone who has
to fit in a downtime schedule, knows that it can be pretty stressful.

Docker, in contrast, lets you test your changes beforehand. If you have built your containers well, that is. I run
hundreds of containers in production, and yet, I can simply use `docker-compose` to launch a miniature version of our
system in my laptop. What's more, since I can launch a miniature version, I can also write integration tests against
the infrastucture. If it's a custom nginx config, I can run a separate container that simply fires `curl` commands
against the webserver. Or if it's a full web application, I can write full integration tests with whatever test suite
I chose.

## Docker & Co isn't perfect

Look, I'll be the first to admit that Docker isn't perfect. Many of the open source / public images available are of
unbelievably poor quality, and running Docker & Co in production is still very very hard. Many would tell you that
Kubernetes is easy to set up... well, I've talked to Google and Amazon ops engineers who would disagree. Containers
have a long road ahead of them until they become a viable, every day tool.

However, that doesn't change the fact that for me and many others Docker has reduced the amount of work needed for CM
and the risk from updates to a point where I can safely update my servers 5-8 times a day. Granted, it takes some 
doing, but it can make ops much less painful.

## TL;DR

Docker gives you the following:

1. Documented infrastructure
2. Reproducibe builds
3. Less chance for a special snowflake server
4. Frequent and easier updates
5. Easier to test changes

However, it has some problems:

1. Poor quality of public images
2. Difficult / unstable production-level tools
3. You still need configuration management for your host machine

And no, it's not about devs can now do ops work, you still need sysadmins. Sorry.
