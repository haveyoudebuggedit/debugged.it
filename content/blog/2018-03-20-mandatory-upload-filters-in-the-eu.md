---
slug: mandatory-upload-filters-in-the-eu
guid: https://pasztor.at/blog/mandatory-upload-filters-in-the-eu/
authors:
- janos
categories:
- Technology
date: "2018-03-20T00:00:00Z"
publishDate: "2018-03-20T00:00:00Z"
summary: A new EU proposal would mandate all online platforms to institute upload
  filters. Let's talk about that.
images:
- posts/mandatory-upload-filters.png
preview: posts/mandatory-upload-filters.jpg
tags:
- Law
title: Mandatory upload filters in the EU?
---

I don't normally talk about law and social issues on this blog. Partially, because **I'm not a lawyer**, I never studied 
law and I may have no idea what I'm talking about. Please send me a [pull request on GitHub to fix any potential mistakes](https://github.com/janoszen/pasztor.at).
But this time is an exception to my usual content, as it affects us all.

An EU proposal earmarked [COM(2016)593](http://eur-lex.europa.eu/legal-content/EN/ALL/?uri=COM:2016:593:FIN) would
institute a requirement for all online platforms to implement an upload filter. To be fair, this proposal is still being
negotiated, but the fact that it even exists is very worrying. 

So what does the proposal say? Let's see:

> ## CHAPTER 2
> 
> Certain uses of protected content by online services
>   
> ### Article 13
>
> Use of protected content by information society service providers storing and giving access to large amounts of works
> and other subject-matter uploaded by their users
> 
> 1. Information society service providers that store and provide to the public access to large amounts of works or
>    other subject-matter uploaded by their users shall, in cooperation with rightholders, take measures to ensure the
>    functioning of agreements concluded with rightholders for the use of their works or other subject-matter or to
>    prevent the availability on their services of works or other subject-matter identified by rightholders through the
>    cooperation with the service providers. Those measures, such as the use of effective content recognition
>    technologies, shall be appropriate and proportionate. The service providers shall provide rightholders with
>    adequate information on the functioning and the deployment of the measures, as well as, when relevant, adequate
>    reporting on the recognition and use of the works and other subject-matter.

OK, that's a mouthful. Let me break it down for you. Assuming this law takes effect, if you run an online service that
deals with user generated content (a.k.a. provides public access to large amounts of works) you will have to implement
an upload filter for copyrighted works.

How would this upload filter work? Well, it doesn't say. Where does the information on what is and isn't copyrighted
come from? It doesn't say. But I'm getting ahead of myself, let's go through step by step.

## How does an upload filter work?

When a user uploads a piece of content to a platform, that piece of content must be checked against a known set of
copyrighted materials. Depending on how data-intensive said content is and how many servers the platform operates for
this purpose, this process can take anywhere between a few milliseconds to hours, or even days in case of an overload.

Since the proposal requires the filter to act on upload (“prevent” being the word used, not “remove”), the content has
to be on hold for the duration of the check. (This may, in fact, disable live streamed content all together.) The
trivial approach to content filtering would be something an 11 year old would think of. Let’s take a text document as an
example, say a book. In this book the algorithm would take the content sentence by sentence and if any of those
sentences are found, it would mark the content as copyrighted.

However, here we encounter our first problem. Let’s take a look at the word “apple”. In “normal” (UTF-8) encoding a
computer would see this word as the following numbers:

<figure>
<table>
<tr><th>a</th><th>p</th><th>p</th><th>l</th><th>e</th></tr>
<tr><td>97</td><td>112</td><td>112</td><td>108</td><td>101</td></tr>
</table>
</figure>

However, I could use similar-looking cyrillic letters to obfuscate most the word:

<figure>
<table>
<tr><th>а</th><th>р</th><th>р</th><th>l</th><th>е</th></tr>
<tr><td>208 176</td><td>209 128</td><td>209 128</td><td>108</td><td>208 181</td></tr>
</table>
</figure>

As you can see, the two representations are wildly different to a computer, while a human can still read the content as
intended. (Unless you are visually impaired and are using a screen reader.) This sort of obfuscation has been common
place among illegal distributors of copyrighted content to avoid detection.

It gets even more complex when talking about audio-visual content, since in order to fit in modern-day bandwidth limits
or on storage devices, a video is highly compressed.

Video compression is a lossy process since it removes all the parts humans can't usually see, and sometimes even
the parts humans can see. This results in the same video being very different if encoded by two different people with 
different settings.

So, instead of simply doing a comparison, the upload filter must now employ visual recognition to detect similarities 
among videos. Visual recognition, however, is a highly resource intensive process and is more often than not very error
prone.

Going further, an upload filter cannot differentiate between legal uses of copyrighted works, commonly known as "fair
use", such as for criticism, news reporting, etc. and actual illegal copies. To make matters worse, the
legality of unlicensed use of copyrighted material varies from EU member state to member state.

## Costs of an upload filter

Now you may think to yourself, ok, how much does it really cost? Let's do some numbers. At the time of writing
[IMDB](http://www.imdb.com/) film database lists about 4.7 million audiovisual works. Let's assume for the sake of
argument that each of these works is, on average, 400 MB in size, which about one tenth of a DVD quality movie. The
total storage required would then be around 2 petabytes. That's one thousand disks with 2 TB of storage space each.
Even if we get the size down to just 40 MB of fingerprint data (I'm being *very* optimistic here), that's still one
hundred high speed disks, plus their associated servers that you put them into.

I hope you can see the scale of what's required of the content providers here. This may work for YouTube but definitely
won't work for a startup. But it gets worse.

You see, most of these works aren't even properly digitalized. They are simply not available, or if they are, the rights
holders may ask for money to obtain them. What's stopping them? Are we going to have to buy 4 million DVDs?

I'm not even going into details like how fast you need to access the data to match the content, or that there are no
algorithms available on the market to do this. (Even YouTubes Content ID messes up all the time.) You can see the 
insanity of this law unless we are talking of the largest of the large companies.

But let’s not go that far, let’s take a look how much money a third party platform would cost. Although I have not been
able to find such a platform, we can take a look at [Amazon Rekognition](https://aws.amazon.com/rekognition/pricing/)
as an example that’s close enough. This system does a much simpler job of simply tracking an object in a video (e.g.
face detection).

Video analysis on this platform costs 0.1$ per minute of video (0.12$ for live video) at the time of writing. If one
creator streams about 2 hours of video a day (an average metric for video game streams for example), each creator would
cost the platform over 300 USD a month (2x60x30x0.1)

The metrics platform [NoScope](http://noscope.io/) claims that there are 200.000 active streamers on
[Twitch.tv](https://www.twitch.tv/). Assuming Twitch had to pay this amount of money for each active streamer, they
would end up with a monthly bill of 72 million USD, or over 800 million USD per year. To put this into context,
[Amazon paid 970 million USD for Twitch in 2014](https://en.wikipedia.org/wiki/Twitch.tv#Amazon_subsidiary_(2014%E2%80%93present)).
In-house systems can, of course, make this vastly cheaper, but building in-house systems can only be afforded by
companies with large engineering teams, and will still constitute a significant operational expense.

You may, of course, say that the law requires a “proportionate” deployment of such scanning techniques, so a startup
cannot be expected to implement these, but let me be the bearer of bad news: I once heard a judge define “proportionate”
as being all the money a company makes right up until the point where they go bankrupt. Until that point, everything is
“proportionate”. So instead of hiring new people, a startup has to invest in deploying a content filter.

## Not all content is created equal

So far we have (mostly) been talking about video content, which granted, is probably the most resource-intensive content
to match one can imagine. But there's a different problem too.

What about source code, for example?

First of all, getting access to proprietary technology for fingerprinting purposes, such as the iOS kernel, would be
nigh on impossible. Yet, the proposal requires the deployment of such a filter on, say, on a Git Service Provider, since
it is not the amount of infringement done on a platform that counts, but the amount of copyrighted content on the
platform itself.

What's copyrighted you ask? Well, almost all works of art or creativity made in the last 70 years are subject to
copyright. Even if you put it up under a very liberal license, it is still copyrighted.

So, let's set the access issue aside for a moment. What were to happen, if a Big Bad Company were to use an open source
library of a Small Indie Developer, and then put their whole source code into the content filter of the service
provider?

Well, one would have to assume that the Git Service Provider would block the upload of the Small Indie Developer, since
the Big Bad Corporation has access to the content filter and the Small Indie Developer doesn't. So now we have another
[left-pad situation](https://www.theregister.co.uk/2016/03/23/npm_left_pad_chaos/).

## Recourse mechanisms?

This brings us to the next point in the law. It is written that:

> Member States shall ensure that the service providers referred to in paragraph 1 put in place complaints and
> redress mechanisms that are available to users in case of disputes over the application of the measures referred to
> in paragraph 1.

So, according to this law, each EU member state has to work out what kind of recourse mechanism they will require. We 
should move away from having a patchwork regulation across Europe, not create more of it. This way we will end up with
a bunch of rules, each different, many of which are badly written and give service providers a loophole not to implement
recourse mechanisms properly.

Back to our story, the Small Indie Developer now writes the Git Service Provider an e-mail, explaining the situation.
The Git Service Provider now has to decide: do they believe the Small Indie Developer or not? If they do not and they
are wrong... well, tough luck.

If they *do* believe the developer and the content *was* actually copyright infringement, the Git Service Provider is 
now on the hook for allowing pirated content on their platform. The safe harbor exception is no longer valid, since they
have actually seen the content and have decided it not to be infringement.

With that in mind, which provider would put themselves on the line for a small developer? Or other types of content?

## Censorship, legalized

Let's look at a different scenario. What happens if a big media conglomerate were to abuse this system to censor
criticism. Like what [Universal Media Group](https://arstechnica.com/tech-policy/2011/12/umg-we-have-the-right-to-block-or-remove-youtube-videos/)
did in 2011.

As far as I can tell, there is no penalty codified into the law. A rights holder could filter content it doesn't even 
own the rights to with impunity. Enforcing an individual creators rights would take a lengthy legal battle most small
time creators could probably not even afford.

## Who even is a service provider?

All that being said, who would be required to implement an upload filter? The proposal doesn't explain. To my webhosting
company could be considered to be a service provider since I, the user, can upload large amounts of works. My own
works, but works nonetheless.

This means, the following could all be considered service providers:

- Video and audio platforms (e.g. YouTube)
- Blog hosting platforms (e.g. Medium)
- File storage services (e.g. Dropbox)
- Code storage services (e.g. GitHub)
- Web hosting and infrastructure as a service providers (e.g. Amazon Web Services ??? How would that even work?)
- Any site with a large enough comment section to be considered a *large amount of works*. Good thing I don't have a
  comment section I guess...

Should we start coding exceptions into the law? I don't think so. As it stands, this regulation is way too broad and
will seriously hamper innovation in Europe.

## Conclusion

The law makers proposing this are probably thinking of large companies like Google and Facebook. What they are not
thinking about are small companies (and there are a lot of those) that will have to spend most of their engineering
budged on this monstrosity of a law.

So, what can you do? First of all, inform yourself. I'm not a lawyer, policy maker or anything of the sort. [Read the
actual proposal](http://eur-lex.europa.eu/legal-content/EN/ALL/?uri=COM:2016:593:FIN), and if I may suggest, the
following resources:

- [EU wants to require platforms to filter uploaded content (including code)](https://blog.github.com/2018-03-14-eu-proposal-upload-filters-code/)
- [Julia Reda (Greens/EFA, Germany): Green light for upload filters: EU Parliament’s copyright rapporteur has learned nothing from year-long debate](https://juliareda.eu/2018/02/voss-upload-filters/)
- [Julia Reda (Greens/EFA, Germany): When filters fail: These cases show we can’t trust algorithms to clean up the internet](https://juliareda.eu/2017/09/when-filters-fail/)
  

Then, if you have gained sufficient insight into the topic, contact your [Members of EU Parliment](http://www.europarl.europa.eu/committees/en/juri/members.html), [Council Members](http://europa.eu/whoiswho/public/index.cfm?fuseaction=idea.hierarchy&nodeID=6623&lang=en) and [Commissioners](https://ec.europa.eu/commission/commissioners/2014-2019_en)
and relay to them that you want the upload filters from this proposal gone.

Also, this article is not subject to the usual [copyright rules](/terms) on this site, feel free to use any parts of the
text on this page in your letters, blog posts or wherever you please.
