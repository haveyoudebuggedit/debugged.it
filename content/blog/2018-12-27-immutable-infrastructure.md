---
slug: immutable-infrastructure
guid: https://pasztor.at/blog/immutable-infrastructure/
authors:
- janos
categories:
- Technology
date: "2018-12-27T00:00:00Z"
publishDate: "2018-12-27T00:00:00Z"
summary: Immutability is an important concept I talked about before. But how do we apply it to infrastructure?
images:
- posts/immutable-infrastructure.png
preview: posts/immutable-infrastructure.jpg
tags:
- DevOps
title: Immutable Infrastructure?
---

OK, so you built a new web application and now you want to get it running. You rent a server at the hosting provider of
your choice, SSH in and get to work. You install your nginx, PHP, MySQL and what not, all by hand.

Now, what happens if you need to pull up a development environment. You need to do all of the work again. If you need to
reinstall the production environment, the same.

What about updates? Every time there is a security hole, you need to update your environment. If it fails, you have an
outage.

## Configuration management

I think from the examples I brought, I think it is clear that installing things by hand is not sustainable. We need a
solution.

About 5-10 years ago people doing ops thought the solution would be *configuration management*. Basically, they wrote
a program that installed the servers. It didn't matter if that was a shell script,
[Puppet](https://puppet.com/) or [Ansible](https://www.ansible.com/), the point of each of these was that when
a server needed to be installed or updated, this software would run on the server and *&ldquo;do the deed&rdquo;*

On the upside, now you had documentation of your infrastructure. Simply reading the code would tell you how the servers
were set up.

On the downside, however, running an update script could still mess up your production environment and cause an outage.
Configuration management would not, in and of itself, fix the issue of broken updates. You still had to keep a massive
amount of machines around simply to test your configuration management script. 

## Enter: the Cloud

In the mean time, discarded by many sysadmins, the cloud made its way into the public consciousness. The benefit
was very pronounced: instead of having to wait 3-6 months to order a new server, you could simply spin up a machine,
test what you want, and shut it down afterwards, *only paying for what you used*.

This paved the way for a radically new approach: *don't update servers*. Yes, at all. If you need an updated server,
simply create a new one, deploy your code there, switch over the traffic and shut down the old one.

Needless to say, to do this you need configuration management tools more than ever. However, this approach means that
you can make sure that you have an actually working environment before you move the traffic over.

## Docker

Docker allows you to create a script, a recipe of sorts how to build a virtual machine image. Keep in mind that it is
not a true virtual machine most of the time, and it usually only runs one program, but it is usually an archive copy of
an operating system. These images can be pushed to a storage called Docker registries, and can be downloaded on other
machines to run them.

In addition, these Docker containers can also accept various runtime configurations, such as environment variables,
which makes them reusable.   

This, of course, made this process a lot simpler, because the base machines basically only need to run Docker and
contain a couple of SSH keys. You wouldn't need to automate the whole installation process using, for example, Ansible,
just the basics. The rest of the software stack would be neatly packed in a Docker container and could be shipped to the
production environment after it has been thoroughly tested.

Needless to say, to get the most of it, you also need to adhere to the principle of *immutability* in your Docker
containers too. You shouldn't update a container, simply throw it away and spin up the new version. If it fails,
go back to the old image.

## Conclusion

Don't update. Version and replace. If the new version fails, go back to the old one. This is, of course, easier said
than done, but in the long run it pays off.

[**Curious how it works in practice? Click here!**](/blog/immutable-infra-1)