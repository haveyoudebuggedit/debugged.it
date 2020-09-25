---
slug: reproducability
guid: https://pasztor.at/blog/reproducability/
authors:
- janos
categories:
- Technology
date: "2018-04-21T00:00:00Z"
publishDate: "2018-04-21T00:00:00Z"
summary: Build environments, server setups should be reproducible, even 6 months down
  the line. How does this work?
images:
- posts/reproducability.png
preview: posts/reproducability.jpg
tags:
- DevOps
- Software Development
title: Reproducability
---

Let's say you are developing a software project. If you are in NodeJS, you specify your dependencies in your
`package.json`, in PHP you would use `composer.json`, etc. Everything is fine, a few weeks later you ship the product
and you're done with it.

The same goes for the *server setup*. You rent a virtual machine, install everything the old fashioned way
(`apt-get install`) and you even take care to enable automatic updates.

So far so good. Project being done, you move on to the next one. Half a year later the client resurfaces and wants some
changes. You check out the project from git and attempt to start the development environment and there comes the first
problem: the project won't run.

As you debug the issue, you realize that the developers of some of the dependencies have introduced breaking changes
into their libraries. One of the libraries no longer even exists. Now you are left with having to hunt down the changes
to those libraries and find a replacement for the one that's gone.

The same happens on the server side: various bits and pieces of the server setups have been passed down as tribal
knowledge, you have no documentation of how it was actually set up, the little text that has been written down as
*&ldquo;documentation&rdquo;* is not really useful in trying to set up a second server.

## Fixed versions for reproducible builds

[Maven Central](https://search.maven.org/), the main repository for Java libraries, has long adopted the policy of using
fixed versions instead of specifying version ranges. While a PHP dependency may specify `somelib` version `1.0.*` or
similar, Maven dependencies are typically written as a fixed version, such as `1.0.23`.

The reasoning behind this is that builds of your program should be *reproducible*. Every time you build, no matter how
far down the line it is, you should be able to do so and the same code being built should yield the same results. That
is also why once a version has been sent to Maven Central, that version can no longer be overwritten or removed, save
for a few rare circumstances.

On the flip side of the coin, however, you do need a utility that tells you if a certain library has a newer version
available and tells you to update your dependencies to fix any potential vulnerabilities. Ideally, you would integrate
such utilities in your automated build system and let it send you an e-mail if a new library version should be looked
at.

## Reproducible infrastructure

The same principle can &mdash; and indeed should &mdash; be applied to infrastructure. If you have a &ldquo;central
development server&rdquo; that took a long time to set up, or even worse, your production infrastructure has been set
up in a way that you cannot easily replicate, you may end up having a heap of unexpected work on your plate. A seemingly
trivial task can evolve into a weeks worth of pain and suffering just because you had to rebuild the *special snowflake*
dev server by hand.

In my opinion, if you need a central dev server because setting up individual dev environments for your developers is 
too hard, you are doing something wrong. Either it is impossible to set up the servers and utilities quickly in an 
automated fashion, your test dataset is not easy to load, or something other funkiness is afoot.

Similarly, in a prod environment: how would you test a major change to your system if you can't quickly set up a
production-like environment in an automated fashion? Many companies announce a downtime and then just
*&ldquo;wing it&rdquo;*. Usually the scheduled downtime stretches to unscheduled lengths because something
unexpected always happens.

If, on the other hand, your infrastructure was built with the expectation to run a single command, load your data
backups and have a fully functional environment, testing changes becomes incredibly easy. That is by [Docker is such a
powerful tool if used correctly.](/blog/why-docker-matters)

## Reproducability is documentation

Having a reproducible system usually means that there is very little need for actual, hand-written documentation. The
instruction set runnable by your build system, your `Dockerfile` or your Ansible configuration
*becomes your documentation*.

In other words, you now have **executable documentation**. There is no way you will forget to update it or miss a
detail, since it won't work if you do. The only thing you need to do from here on is to write high level, conceptual
overviews about your system and business-level documents.
