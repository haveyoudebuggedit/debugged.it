---
slug: lxc-vs-docker
guid: https://pasztor.at/blog/lxc-vs-docker/
authors:
- janos
categories:
- Technology
date: "2018-03-08T00:00:00Z"
publishDate: "2018-03-08T00:00:00Z"
summary: LXC is the older of the two, but how do they compare? What's the difference? Which one should you choose for your next project?
images:
- posts/lxc-vs-docker.png
preview: posts/lxc-vs-docker.jpg
tags:
- DevOps
- Containers
title: LXC vs Docker
---

With the rising popularity of Docker the question presents itself: why Docker? Why not the much older, and better
understood LXC, what's the difference? Why pick one over the other?

What you have to understand that there is a fundamental, **philosophical difference** between LXC and Docker.

Containerization itself has a long history in the BSD world, but was introduced to the mainstream Linux community in the
form of [OpenVZ](https://openvz.org/). Before OpenVZ the Linux kernel had no means to create any sort of
containerization, apart from the `chroot()` functionality that allowed a process to be run using a different view of the
filesystem.

At first, the OpenVZ team developed patches for the Linux kernel to implement these features, which then
made their way into the Linux kernel thanks to the efforts of the OpenVZ team, IBM, Google, and others.
These features in the Linux kernel, which I have written about in [my article about
Dockers internals](/blog/under-the-hood-of-docker), are utilized by the LXC, Linux-Vserver, and indeed Docker to create
containers.

**LXC** itself is a spiritual successor of OpenVZ. While OpenVZ is still around, mostly in the Redhat world with older
kernels, LXC is the tool of choice for many who who wish to **run a full operating system in a container**. As such, LXC
is more akin to true VMs and has to be handled much in the same way: software is installed by hand, updates have to be
run, configuration management is much needed for keeping the madness at bay.

**Docker** on the other hand adopts a much different approach. Instead of running a VM-like container with a full
software stack, including an init system, a syslog server, cron daemon and all the other stuff that one may have, it
**is built for running one application**.

Not only that, but the recipe to create the environment for that application is an executable piece of documentation
itself, called the `Dockerfile`. Docker uses this `Dockerfile` to build an image (basically a bunch of tar files) of the
application and the libraries it needs, then ships these to a production environment. This image is then run, and some
kernel magic (via OverlayFS, AUFS or DeviceMapper) takes care of the files that are written temporarily as a result of
the container running.

When an update is needed, one does not SSH into a Docker container and install the updates by hand either. Instead of
updating the OS, a new image is built with the updated software, and the old one is simply thrown away. Persistent data
can be handled using *volumes*, and the application code is, most of the time, shipped inside the Docker image.

This alleviates a lot of the problems with classic system operations, where updates would usually be a major pain 
and configuration management is a point of contention in and of itself. Updates can be tested in a local environment,
then shipped to production. I've written about this before in my article about
[why Docker matters](/blog/why-docker-matters).

Does this then mean that *Docker is superior?* Well, no. First of all, not all software is suitable for running inside 
a container that can be thrown away at a moments notice. Some software requires human interaction to install, and some
software does not deal well with being stopped with only a few seconds notice. Runtime configuration can also be a major
problem, if, for example, it is stored in a database that is not easily reconfigurable via a shell script.

Probably by far the most problematic item in Dockers sizable laundry list is the lack of easy to use orchestration
tools. Sure, almost every cloud provider offers Kubernetes support nowadays, but running it on your own hardware, or
even your own VMs is a daunting prospect when getting started because of its insane complexity. Docker Swarm would be
a great tool, but it has, in my experience, been unstable to use.

The only moderately easy to use tool we are left with is [docker-compose](https://docs.docker.com/compose/), which is
little more than a glorified bunch of scripts used to describe multiple containers. Its two main drawbacks are, that
1) it has no running daemon and relies heavily on Docker to do the right thing, and 2) that it isn't really built for
coordinating containers across multiple machines (except in conjunction with Docker Swarm).

That being said, running throw-away containers that I can test and update any time beats the hell out of having to write
several thousand, or even tens of thousands of lines of code for configuration management.

I guess the moral of the story is: pick your poison. At this time Docker is still very young, so LXC is a valid and
viable solution if you want to stick with the classic ops model for a little longer.

{{% warning %}}
**Common misconception**: Docker does not use LXC. In the beginning Docker used the LXC command line tools to run containers, but that is no longer the case. Both Docker and LXC use the containerization features in the Linux kernel, but are independent of each other. You can read more about this topic in my article [Under the hood of Docker](/blog/under-the-hood-of-docker)
{{% warning %}}

{{% tip %}}
I would like to thank [Kir Kolyshkin](https://twitter.com/kolyshkin) for the help in Linux containerization history.
{{% /tip %}}
