---
slug: why-people-want-kubernetes
guid: https://pasztor.at/blog/why-people-want-kubernetes/
authors:
- janos
categories:
- Technology
date: "2019-01-23T00:00:00Z"
publishDate: "2019-01-23T00:00:00Z"
summary: I've previously argued that running Kubernetes is hard. Why do people still want it? Let's look past the hype train and take a gander through the valuable things k8s provides.
images:
- posts/why-kubernetes.png
preview: posts/why-kubernetes.jpg
tags:
- DevOps
- Containers
title: Why people want Kubernetes?
---

In a [previous article I argued that Kubernetes is extremely hard to run](/blog/kubernetes-is-hard). Many people have
rightfully asked why someone would even bother with using Kubernetes? Isn't it all just a hype train that people 
have jumped on and are heading right for that proverbial cliff?

Sure enough, Kubernetes (k8s) is on the top of the
[Gartner Hype Cycle](https://www.gartner.com/en/research/methodologies/gartner-hype-cycle). Many newly created
*&ldquo;DevOps&rdquo;* teams are facing inevitable disappointment. (You know, the ones that have been rebranded DevOps
because it sounds cooler.)

Nevertheless, Kubernetes does provide tremendous value if used right. Let's take a gander what these values are and
why you might want to care about them.

## Reproducability

Reproducability is again a topic [I have written about before on this blog](/blog/reproducability). It is the idea that
even if I delete my complete production environment and just back up my data, I can still reproduce all my machines,
configuration, and whatever is needed for my software, restore my backups, and be right back where I was.

This is, of course, great for disaster recovery. Assuming you followed your cloud providers guidelines and stored your
backups and logs in a separate cloud account, even the compromise of your main cloud account lets you restore
everything.

If you stick to the concept of reproducability, you never used the web interface for your cloud account, but instead
used tools like [Ansible](https://www.ansible.com/) or [Terraform](https://www.terraform.io/) to create everything.
*The documentation of how your cloud account should be set up is literally in your git repository* in an executable
form.

Needless to say, if you need to code everything yourself in Ansible or Terraform, the cost of development for even
the simplest web applications is going to exceed any reasonable time frame. Before the container days I used to write
thousands upon thousands upon thousands of lines of configuration management code. I had a [Puppet](https://puppet.com/)
repository that was about the same size as the production code that was deployed in the infrastructure. Everything
had to be coded, from a mount point to a firewall rule.

Kubernetes makes the whole reproducability issue simpler. Love it or hate it, everything can be described in relatively
simple YAML files, from the mount points to the firewall rules, but even the health checks for each individual service.
It takes care of managing the DNS records needed to connect services and much, much more.

But we can go one step further. Instead of having *one* infrastructure, we can use the same recipe to create *multiple*
infrastructures. Need a new prod-like setup to demo the code in a branch not yet ready to be deployed? Sure, take your 
YAML files and deploy it into a separate namespace on your Kubernetes cluster.

Sounds easy, right? Well, it isn't. It still isn't. In order to have this kind of flexibility, you still need to do
things right. Your containers need to be built in such a way that you can configure them dynamically. Your YAML files
need to be written with the expectation that there will be multiple environments. The deployment process needs to be
integrated into your CI system so pulling up an environment is easy. After all, if it's too hard or time consuming,
nobody will do it.

## Immutability

The other important concept that has gained traction lately is immutability in terms of infrastructure. The idea here
is that you never update anything.

*Wait, what?*

Yes. You never update anything. Servers, containers, nothing. Instead, you *replace* them.

Let's say you have a server that's running a webserver. Traditionally you would schedule a downtime to upgrade the 
software, starting at midnight to 6 AM in the morning. Already sounds fun, right?

You run the updates and some time at 2 AM in the morning you hit a snag. Something in the new software version has
changed. The rest of the night is spent with furiously debugging, reading the docs, trying to fix the configuration, and
at 8 AM, 2 hours past the announced end of the downtime, you finally manage to get the service back online.

Instead, let's do the following: let's not update our server at all. When we need to update something, we install a new
server with the updated software package on it. Before we put it into production, we test it if it works. Ideally our 
software has some sort of test suite that enables us to do the appropriate compatibility checks, and when everything is
done, we switch over to the new server.

This, of course, is again easier said than done. First of all, you need a spare server. In the time when we were still
using physical servers this was a budget-issue as well as a technology one: we had to have the same type of server
on hand to deploy the infrastructure on. Virtual machines and cloud providers, of course, make this much easier.

Second, we also need automation tools to install the servers. After all, who would do weekly software updates by 
manually installing servers? You'd have to be a masochist to do that.

This, again, is something Kubernetes excells at. The virtual or physical machines running under K8 become much simpler
to install because the concept is that you should never ever have to SSH into them. They &ldquo;just&rdquo; need to
run the Kubelet on top of a very bare bones Linux. This is easier to install in an automated fashion than a full Linux
with lots of users and a full web stack. It can be as simple as putting a reasonably sized shell script in the
[cloud-init](https://cloudinit.readthedocs.io/en/latest/) field of your VM creation.

The containers running on top of Kubernetes are themselves immutable if done correctly. The core concept of containers
is that instead of upgrading them, we replace them. If we have problems with the new version, we can (hopefully) 
roll back to the previous version of the container.

The concept of immutability, of course, starts to break apart when the container needs to store persistent data. When
upgrading, for example, a database, the new version may not be compatible with the old one. After upgrading one database
server in a cluster it may not be able to re-join the cluster of older nodes. You may also not be able to roll back to 
the old version if the data storage format has been changed to the new version.

This just goes to showcase that containers or Kubernetes are not the end-all-be-all of operations.

## Easier deployment

Deployment on servers has also been a tricky proposition. If we wanted to deploy a PHP application in the old days the
deployment script would need to upload the new version of the application into a separate directory, and then switch
a symlink on all servers. It would also need empty caches, populate the databases, and so on.

With containers this whole process becomes simpler. The code of the application is *baked* into the container image and
launching the container takes care of the initialization for the application. Caches are purged naturally as the 
container is replaced.

Kubernetes takes this one step further. Being an API-based system, the deployment can be done with a simple `curl` call.
In contrast to other solutions like Docker Swarm Kubernetes also has a sophisticated permission system to make sure
that, for example, no privileged containers can be launched that could compromise the entire Kubernetes cluster.

## Networking abstraction

One more feature that is a huge quality of life improvement in Kubernetes is the networking abstraction. In the old days
configuring a firewall would be a matter of writing an `iptables` script. Iptables itself, however, is a tool that even
most sysadmins got wrong. Over time the Linux kernel improved, but it was still easy to get wrong.

More importantly, however, app developers rarely had the permissions or know-how to change these rules. New ports and IP
address unblocks would often need to be requested from sysadmins, which is why many sysadmins opted to simply allow 
everything on the internal network. This would make attacks like [SSRF](/blog/what-is-ssrf) easier.

Kubernetes with its overlay networks and network policies hides away the complexity of the physical networks and makes
the firewall rules deployable via a simple API call. In other words, the firewall rules can be stored in the git repo
along with the other configuration of how the application should be run.

This makes some sysadmins uncomfortable, but I would argue that a maintained firewall is better than an unmaintained. 
Instead of ensuring compliance to policies by restricting the firewall configuration to privileged employees, active 
scans and Kubernetes audit tools should be used to ensure security.

## You (probably) don't have Google-scale problems

We could probably write a book about how great Kubernetes is, but the point is: there are valid reasons behind wanting
to use Kubernetes beyond the hype train. Discarding it as a valuable tool just because there is a hype going in is
like sticking your fingers into your ears and saying &ldquo;lalalalala&rdquo; very loudly.

Kubernetes has been built by Google to address Google-scale problems. As I've written before,
[Kubernetes is hard](/blog/kubernetes-is-hard) and may not be for everyone. The concepts behind using k8s, however, are
valuable and should at least be given some thought when deploying a new infrastructure.
