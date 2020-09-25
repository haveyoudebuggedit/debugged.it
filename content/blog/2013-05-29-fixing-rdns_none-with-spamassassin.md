---
slug: fixing-rdns_none-with-spamassassin
guid: https://pasztor.at/blog/fixing-rdns_none-with-spamassassin/
authors:
- janos
categories:
- Technology
date: "2013-05-29T00:00:00Z"
publishDate: "2013-05-29T00:00:00Z"
summary: When dealing with SpamAssassin and Exim, one may often encounter a mysterious
  RDNS_NONE
tags:
- DevOps
- Exim
title: Fixing RDNS_NONE with Spamassassin
---

After [filtering spam with Exim](/blog/filtering-spam-with-exim-only/), I wanted to add [Spamassassin](http://spamassassin.apache.org/) to do content based filtering. While testing the spam filtering, I ran into a bit of an issue: I encountered a spam score factor in every single e-mail: `RDNS_NONE` with the score of 1.3.

Doing a [quick Google](http://www.google.com/?q=RDNS_NONE) turns up some [less-than-useful documentation](http://wiki.apache.org/spamassassin/Rules/RDNS_NONE) pages and a lot of people with the same problem, yet no solution. So let’s go hunting…

## Theory

In theory Spamassassin checks all `Received` headers and looks for IP addresses and reverse DNS checks added by the mail server. So if you have `RDNS_NONE` in all your mail, it means that your mail server doesn’t check the reverse DNS properly. Except _my mail server did_.

## Looking at the code

Since obviously this whole issue is a bit underdocumented, I went to look at the SA code. (For the record: I hate Perl.) After `grep`‘ing around in the code, I became suspicious of `/usr/share/perl5/Mail/Spamassassin/Message/Metadata/Received.pm`. This file basically contains a big honk of regular expressions for all kinds of mail servers.

So, **if you have a custom `Received` header, it won’t work**.

## Fixing the issue

If your mail server is set up properly, **you shouldn’t accept mail from servers without proper reverse DNS**. You may also want to check my [previous blogpost on filtering spam with Exim](/blog/filtering-spam-with-exim-only/).

In other words you may not even need this rule. So to disable it, enter the following line in `/etc/spamassassin/local.cf`:

```
score RDNS_NONE 0
```

**Only do this if you reject messages form servers without a reverse DNS on your mailserver!** This will effectively disable this rule from running.

Do you have a better solution? Do you have something to add? Leave a comment below!
