---
slug: kubernetes-is-hard
guid: https://pasztor.at/blog/kubernetes-is-hard/
authors:
- janos
categories:
- Technology
date: "2018-12-04T00:00:00Z"
publishDate: "2018-12-04T00:00:00Z"
summary: Kubernetes won the container wars... allegedly. However, Kubernetes is still
  hard and causing a lot of grief.
images:
- posts/kubernetes-is-hard.png
preview: posts/kubernetes-is-hard.jpg
tags:
- DevOps
- Containers
title: Kubernetes is hard
---

I guess I should give a little preface to this article. [Kubernetes](https://kubernetes.io/) is the new runtime for many
applications and when used correctly it can be a powerful tool to get complexity out of your development lifecycle.
However, over the past few years I have seen many people and companies trip up on wanting to run their own installation.
Often it stays in an experimental stage and never makes it into production.

## How does Kubernetes work?

In broad strokes, Kubernetes or K8s seems to be very simple. The nodes (machines) you run Kubernetes on are split into
(at least) two types: the master and the workers. The master (or masters), by default, does not run any actual
workload, that's the workers job.

The Kubernetes master includes a component called the API server which provides an API that you can talk to using the
`kubectl`. In addition it also includes a scheduler, which makes decisions on which container is supposed to run where
(*schedules* the containers). The final component is the controller-manager, which is actually a set of multiple
controllers responsible for handling node outages, handling replications, joining services and pods (sets of containers),
and finally dealing with service accounts and API access tokens. All the data is being stored in etcd, which is a
strongly consistent key-value store (with some really cool features). So to sum it up, the master is responsible for 
managing the cluster. No big surprise there.

The worker, on the other hand, is running the actual workloads. To do that it includes, again, a number of components.
First off, it runs the *kubelet*, which is again an API that works with the containers on that node. There's also the
*kube-proxy*, which forwards network connections, *containerd* to run containers, and depending on your configuration
there may be other things such as *kube-dns* or *gVisor*. You will also need some sort of an *overlay network* or an
integration with your underlying networking setup so Kubernetes can manage the network between your pods. 

If you want a more complete overview, I recommend doing Kelsey Hightowers
[Kubernetes - The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).

## Production-ready Kubernetes

This, so far, doesn't sound too bad. Install a couple of programs, configuration, certificates, etc. Don't get me 
wrong, it's still a learning curve, but it's nothing an average sysadmin didn't have to deal with in the past.

However, just simply installing Kubernetes by hand isn't exactly production ready, so let's talk about the steps needed
to get this thing up and running.

First, **installation**. You really want to have some sort of an automated installation. It doesn't matter if it's
Ansible, Terraform or other tools, you want to have it automated. [kops](https://github.com/kubernetes/kops), for
example, helps you with this, but using kops means that you won't know how exactly it is set up and that may cause
issues when you later want to debug something. This automation should be tested, and tested regularly.

Next up, you need to **monitor** your Kubernetes installation. So right away you need something like
[Prometheus](https://prometheus.io/), [Grafana](https://grafana.com/), etc. Do you run it inside of your Kubernetes? If
your Kubernetes has a problem, is your monitoring then down? Or do you run it separately? If yes, then where do you run
it?

Also noteworthy are **backups**. What will you do if your masters crash, the data is unrecoverable and you need to
reprovision all pods on the system? Did you test how long it takes to run all jobs in your CI system again? Do you
have a disaster recovery plan?

Now, since we are talking about the CI system, you need to run a **Docker registry** for your images. This, of course,
you can again do in Kubernetes, but if Kubernetes crashes... you know the drill. The **CI system** is, of course, also 
a concern, as is running your **version control system**. Ideally, isolated from your production environment so that if
that system has an issue, at least you can access your git, re-deploy, etc.

## Data storage

Finally, let's talk about the elephant in the room: **storage**. Kubernetes in and of itself does not provide a storage
solution. Of course, you can mount a directory from the host machine, but that is neither recommended, nor is it simple.

You basically **need** to provide some sort of storage under Kubernetes. [rook](https://rook.io/), for example, makes it 
relatively simple to run [Ceph](https://ceph.com/) as an underlying block storage for your data storage needs, but my
experience with Ceph is that it has a *lot* of knobs that need tuning, so you are by no means out of the woods by simply
hitting next-next-finish.

## Debugging

When talking about Kubernetes with developers, one common pattern came up quite regularly: when using a managed 
Kubernetes, people had problems **debugging** their applications. Even simple problems, such as a container failing to
start, caused confusion.

This, of course, is an education problem. Over the past few decades developers have learned debugging
*&ldquo;classic&rdquo;* setups: reading log files in `/var/log`, etc. But with containers we don't even know which 
server the container is running on, so it presents a paradigm shift.

## The problem: complexity

You may have noticed that I'm skipping over the things cloud providers give you, even if it's not a full managed
Kubernetes. Of course, if you use a managed Kubernetes solution, that's great, and you won't need to deal with any of
this, except for debugging.

Kubernetes has many-many moving parts, and Kubernetes on its own does not provide a full stack of solutions.
[RedHat OpenShift](https://www.openshift.com/), for example, does, but it costs money and you still need to add things
yourself.

Right now Kubernetes is on the hill of the
[Gartner hype cycle](https://www.gartner.com/en/research/methodologies/gartner-hype-cycle), everybody wants it but few
people truly understand it. Over the coming years quite a few companies will have to realize that Kubernetes is not the
silver bullet and figure out how to use it properly and efficiently.

I think, running your own Kubernetes is only worth it if you can afford to dedicate an ops team to the topic of
maintaining the underlying platform for your developers.