---
layout:        post
title:         "Monitoring basics"
date:          "2018-12-31 00:00:00"
categories:    blog
excerpt:       "Monitoring your application is critical. But how do you do it? What are the important things you need to watch out for?"
preview:       /assets/img/monitoring-basics.jpg
fbimage:       /assets/img/monitoring-basics.png
twitterimage:  /assets/img/monitoring-basics.png
googleimage:   /assets/img/monitoring-basics.png
twitter_card:  summary_large_image
tags:          [DevOps, Theory]
sharing:
  twitter:  "" 
  facebook: ""
  linkedin: ""
  patreon:  ""
  discord:  "@everyone"
---

If you ask a sysadmin what the most important thing in running an infrastructure is, they are either going to say
&ldquo;backups&rdquo; or &ldquo;monitoring&rdquo;. Yet, a surprising amount of systems only have rudimentary monitoring
at best.

When I go into a system to do an audit, I usually discover that if they even have a monitoring, they usually run with
some default settings that do no good whatsoever.

So how do we build a monitoring system?

## The questions monitoring answers

Before we dive in, we need to ask ourselves the question: What is the purpose of our monitoring system? What should we
monitor?

The obvious answer I hear from a lot of business owners is: *&ldquo;Everything!&rdquo; But nothing could be further
from the truth.

In my eyes monitoring has two purposes. First of all, it needs to tell you when something happens in the system that
impairs the business. Nobody is wants to get up at 2 AM in the morning for something that doesn't affect the business.

Second, it should help with forecasting. It should tell you about things that will soon become an issue.

Let's look at an example. If your webshop is down, you will want to know about that pretty much immediately. If it's
2 AM you may need to get up and fix it. On the other hand, if a single disk fails and you have a RAID setup, and your
infrastructure is set up in a redundant fashion anyway, that's more of a forecast-type of issue. You should know about
it, but it should not wake you up.

## The trust-issue

This brings us to the next important point about monitoring: *trust*. A lot of monitoring systems are set up in a way
that they alert for absolutely *everything*.

If you get alerts for irrelevant things, or even worse, something that isn't even an issue, you will lose trust in 
your monitoring system. You will stop paying attention, or even disable alerts.

If your monitoring system alerts for something it shouldn't, you need to go fix it, or
[alarm fatigue](https://en.wikipedia.org/wiki/Alarm_fatigue) will set in. 

## Types of monitoring

Monitoring can be done in a variety of ways. When business owners think of monitoring, they think of it from a top-down
perspective: does the system work. They will want to run a check every couple of minutes, checking if the site loads,
if you can do a purchase, etc.

However, these kinds of checks only tell you if a system is working or not, they don't tell you the reason. So you
need to dig deeper. The situation is similar to the difference between integration tests and unit tests: the former
tells you if the system is working, the latter tells you what's wrong with it. 

## Active vs. passive

A different way of looking at monitoring is the source of the data. Systems with low traffic volume will want to go
with an active check: the monitoring software will actively connect to the service and perform a check of its 
functionality, such as load the website. This is great, because it actively checks, even if users are not hitting the
website, but if there is a temporary failure, it may go uncovered if it lasts less than the period between checks.

The other type of checks is passive, for example by collecting data. If you have enough traffic on your website, or
enough purchases on your webshop, you can set up a system that collects data and alerts if a certain number reaches
a high or low point. For example, you could monitor 5xx HTTP response codes. Or you could monitor that there are at
at least 10 purchases per hour.

However, with thresholds like this you need to be careful. For example, you will want to set a different threshold 
during the night than during the day, otherwise you will find that you get lots of false alarms, or on the flip side,
you don't discover issues quickly enough.

## Backups

Now, one important aspect that is often neglected when it comes to monitoring systems is the backups and disaster
recovery of it. If your monitoring system crashes, you are flying blind. You absolutely have to make backups
of your monitoring system, and you have to be able to restore operations quickly in case of a failure.
