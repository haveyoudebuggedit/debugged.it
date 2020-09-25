---
slug: why-your-site-is-sloooooow
guid: https://pasztor.at/blog/why-your-site-is-sloooooow/
authors:
- janos
categories:
- Technology
date: "2018-01-31T00:00:00Z"
publishDate: "2018-01-31T00:00:00Z"
summary: Google Tag Manager is probably the worst thing that happened to the web in
  the last 10 years. Yes, you read me right. Your shiny marketing tools are murdering
  your conversions. Wonder why? Read on.
images:
- posts/heres-why-your-site-is-slow.png
preview: posts/heres-why-your-site-is-slow.jpg
tags:
- Software Development
title: Your marketing tools are killing your site performance
---

## It's not your Wordpress

Many people would have you believe that your Wordpress, full of all those plugins, are the cause of all your miseries
and will make jokes at your expense for using it. Putting the fact aside that they can't recommend a better system,
let's take a look at the performance.

When we are talking about Wordpress performance, you are usually looking at the Time To First Byte metric. That's the
time PHP needs to process your request. Even with the very worst of the worst, a Wordpress loaded with over 30 plugins
I have never seen a TTFB over much more than a second. In contrast to that, your average load time you are unhappy with
is 5-10 seconds. While yes, improving your application rendering performance CAN improve your load time, that's not
the main culprit, not by a long shot. 

## It's (probably) not your hosting company

Again, others would have you believe that your hosting company is the cause of your sorrows. This can sometimes be true,
if they are running overloaded servers or buy cheap bandwidth over poor connections, you usually switch hosting
providers not over technology, but over support.

Again, taking a look at the worst of the worst, the cheapest of the cheap, this accounts for a second or two at best. As
you will see below, there are far bigger factors in the load times than how fast your hosting provider can deliver your
page. 

## It's all your third party JavaScript

A modern sales page has all kinds of goodies: Google Analytics for metrics, HotJar for heatmaps, Intercom for chat,
Facebook for likes, Twitter for a the tweet box, maybe a payment provider and a bunch of other stuff. All of these have
something in common: they come with a little *snippet* that you just paste and they will start to work like magic.

**Snippet**. Sounds small. Lightweight. Not a big deal, right? Well, wrong. One script alone adds just a tiny bit of 
extra to your load time, but scripts can really take a toll once they are combined.

### JavaScript has no multitasking

Remember, I said one script is no big deal? Well, here's the kicker: JavaScript has no real multitasking. It does
what's called *cooperative multitasking*. In other words, as long as these scripts play nice and don't hog the CPU,
you're good. However, if any of your scripts starts clamping down on the resources, all your other scripts will suffer.
And maybe that totally unimportant metrics script you just included is preventing actual, content-generating scripts
from executing.

### Snippets load more scripts

While it may seem so, the snippet you insert is not the only code that needs to run. If you look at the Google
Analytics tracking code carefully, you will see something like this:

```
var scriptElement = document.createElement("script");
```

This means that your little *snippet* is creating a new `<script>` element and inviting its big brother into the page.

And that's not all, sometimes these snippets can lead to a whole host of things being loaded, even complete pages in
iframes.

### Third party services can slow you down

Another thing that can cause severe load issues is when a third party service is slow. The more third party scripts you
include, the more likely you are to suffer when that third party service has an operational issue and is delivering
something slowly. It could be as trivial as YouTube not being able to deliver your video as fast as you'd like,
specifically, because their storage server is having a bad day. Or that the ISP of the visitor throttling YouTube.
(You see why net neutrality is important now?)

## It's your images

A lot of times, especially when working with Wordpress sites, people tend to upload insanely large images and use them
unresized. This means that you could be transfering as much as 10-20 MB over a shitty wireless connection.

So please resize your images to the size you need them at, and also reduce the quality. Many image resizing libraries
and services allow you to shrink images by 50-70% without a visible quality drop.

## It's (probably) your video hosting provider

I've seen a number of cases in the past few weeks where people complaining about poor performance were putting multiple
video embeds into their page. Most video services offer a simple code that contains a single `<iframe>`, but when
you have a couple of these in your page, that can spectacularly backfire.

So what can you do? Well, you need to lazy-load your videos, and that needs a frontend developers work, or if you
are using a CMS like Wordpress, a plugin.

## Why doesn't Pagespeed / ySlow / GTMetrix / Pingdom / ... tell me this?

You see, none of these problems are problems if they stand alone. If you have a page which serves the primary purpose
of showing a video, it is not useful to lazy load that video, for example. The effects, however, are cumulative.

I've seen multiple people wondering why, despite their seemingly OK Pagespeed and ySlow scores, their site is still
slow. My appeal to you is that you please stop taking these as rules and regard them more as guidelines.

Instead, you should investigate WHY your page is slow by using the Network and Performance tabs in your browsers 
Developer Tools. Find out what's keeping the browser from finishing the load and eliminate it.

## Resist the Urge

Google Tag Manager is probably the worst thing that happened to the web in the last 10 years. It makes it incredibly 
easy to include some third party script as a quick fix or to scratch an itch. Its main appeal is that you don't need
a developer to do it.

However, you are also missing out on the feedback a developer might give you on the script you are about to include.
Maybe it's shit, maybe not. Maybe it works nicely with your existing scripts, maybe it doesn't. Slowly, over time,
things can creep in and your page can get slower and slower, leaving you wondering how you ended up here.

Resist the urge to try out a new fancy marketing tool every week that needs yet another JavaScript. Consult with your
developers before using GTM to just "turn it on". Not only does it make your site faster, but a little conservative
thinking can also go a long way towards [keeping your data secure](https://hackernoon.com/im-harvesting-credit-card-numbers-and-passwords-from-your-site-here-s-how-9a8cb347c5b5).

## TL;DR

- A lot of third party JavaScript means slow load times.
- The effects are cumulative.
- Be (very) conservative about embedding third party JavaScript
- Scale down your images
- Lazy-load your videos
- Pagespeed / ySlow are more guidelines than rules and don't give you a complete picture.