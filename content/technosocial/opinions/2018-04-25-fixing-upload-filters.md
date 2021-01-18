---
slug: fixing-upload-filters
guid: https://pasztor.at/blog/fixing-upload-filters/
authors:
- janos
categories:
- Technology
date: "2018-04-25T00:00:00Z"
publishDate: "2018-04-25T00:00:00Z"
summary: 'Here''s a thought experiment: how would I fix the upload filter law?'
images:
- posts/fixing-upload-filters.png
preview: posts/fixing-upload-filters.jpg
tags:
- Law
title: Fixing upload filters...
---

Following a Twitter debate, [I have been challenged](https://twitter.com/netopiaforum/status/988929002462212096) to
propose a better legislation that still includes upload filters. So I shall rise to the occasion, and here's how I would
make upload filters work, both from a technical and legal perspective.

To be very clear: **I still think that upload filters are a bad idea**, and I am also not a lawyer, but hey, dear EU
politicians, if you are going to make upload filters a thing, please feel free to fix it using these ideas. (And 
let me know if you do, too!)

The latest round of documents regarding this issue are [available here](https://www.parlament.gv.at/PAKT/EU/XXVI/EU/01/86/EU_18668/imfname_10803001.pdf)
(look for Article 13).

## Who is a user?

The primary, most fundamental problem of the new proposed legislation is that it does not clearly define who a user is
and in what kind of relationship an upload filter should or shouldn't be used.

As [mentioned previously](/blog/mandatory-upload-filters-in-the-eu), this could have unintended consequences, such as
Amazon Web Services now having to institute upload filters on S3, which is clearly a Business to Business, and
furthermore, automated operation, so creating upload filters make no sense whatsoever.

As such, I think that the term &ldquo;user&rdquo; should be rewritten as &ldquo;anonymous user&rdquo;, and be clearly
defined as a person using an online service, where the name and address is not known or cannot be verified independently
(such as with a credit/debit card payment). This would exempt B2B transactions and other business relationships where
claims against the infringer can be easily enforced.

## Level of Entry

The second thing that should be considered is the level of entry. Small businesses, startups won't have the capabilities
to implement upload filters. Instead of the nebulous phrasing around taking economics into account, there should be
a hard cap, say 10.000 reported infringement cases in the past year, when a platform needs to implement upload filters.

## Making it an open system

Right now, the proposed law puts a tremendous amount of power into the hands of big publishers and copyright holders.
The platform owners are supposed to negotiate with these rights holders to facilitate upload filters. This, by its
nature, excludes small creators, such as yours truly. (Yes, I regularly have to file takedown requests.)

Instead, what should be done, and I realize this is a lot of work, is the creation of a digital, EU-wide copyright
registry. Whoever wants to make their work protected, would have to submit their copyrighted work electronically, and
pay a modest fee to help with the upkeep of the system. Surely the movie industry will have no problems paying 30-some
euros per work, and it will still put it into the reach of small creators like myself.

The copyright registry would then take the copyrighted work and create a fingerprint, based on type (text, image, audio
or video) and platform holders, again, for a very modest fee could submit fingerprints of uploaded content for
verification.

This necessitates that the fingerprinting algorithm also needs to be open and limits the amount of shenanigans that can
be pulled off. When a new type of content is added to the copyright registry, the fingerprinting algorithm for said
content type would have to prove to have only minimal amounts of false positive based on some historic lawsuits around
copyright. If a certain type of content does not have a history of lawsuits around copyright, it should not be added to
the registry.

## Preventing abuse

This brings us to the next question. How do we prevent people from asserting copyright on works that they don't even
own? For example, when [Universal Media Group took down the Megaupload song](https://arstechnica.com/tech-policy/2011/12/umg-we-have-the-right-to-block-or-remove-youtube-videos/).

One way of doing that is imposing stiff penalties. We won't even have to go far for an example, we could easily lift
the penalties section from the soon to be effective GDPR (EU privacy law). Make the penalty up 4% of
the worldwide annual revenue of the prior financial year and that should deter even the largest companies from pulling
a fast one.

## Redress mechanisms

The current proposal also leaves it up to the member states to define redress mechanisms. In my eyes this is also a
fatal mistake, because it will leave the door open for platform owners to incorporate in countries that have the least
requirements in terms of creating effective mechanisms.

In no case should the decision if a piece of content can go up or not be left in the hands of the purported copyright
holder. There are various edge cases, such as timely news reporting, or reportings on the purported copyright holders
shady actions, which would lead to harm to the uploader if the aforementioned copyright holder had the rights and an
extended period of time to delay or block the upload.

In my eyes the US Digital Millenium Copyright Act should be taken as a template. When an upload filter hits a piece
of content uploaded by an anonymous user, they can either leave it be, or file a &ldquo;counter notification&rdquo;
of sorts, where they have to provide and verify their contact details. Their contact details would be kept on file with
the platform and could be requested by a court should a lawsuit be filed.

A temporary injunction / court order could also take the piece of content down pretty quickly if needed, or the law
could allow for an actual takedown to be filed, akin to the DMCA.

Now, an interesting question arises if the user doing the uploading is not inside the EU. Dragging someone from a non-EU
country to court is definitely an expensive proposition, and I also have no solution to that, except for handling EU and
non-EU citizens differently.

## Scrap exceptions

Various proposals for exceptions have been floating around, excepting anything from nonprofits to online marketplaces
from implementing upload filters.
[Online marketplaces? Really?](https://www.theverge.com/2015/11/29/9813780/udemy-pirated-course-copyright-troy-hunt)

If this law is to be effective, it should apply to everyone. If it doesn't, it's a badly written law and should not be
passed.

## Exceptions to copyright

The US DMCA clearly defines a paragraph called fair use (17 U.S.C. § 107). It says:

> Notwithstanding the provisions of sections 106 and 106A, the fair use of a copyrighted work, including such use by
> reproduction in copies or phonorecords or by any other means specified by that section, for purposes such as
> criticism, comment, news reporting, teaching (including multiple copies for classroom use), scholarship, or research,
> is not an infringement of copyright. In determining whether the use made of a work in any particular case is a fair
> use the factors to be considered shall include—
>
> (1) the purpose and character of the use, including whether such use is of a commercial nature or is for nonprofit
> educational purposes;
> (2) the nature of the copyrighted work;
> (3) the amount and substantiality of the portion used in relation to the copyrighted work as a whole; and
> (4) the effect of the use upon the potential market for or value of the copyrighted work.
>
> The fact that a work is unpublished shall not itself bar a finding of fair use if such finding is made upon
> consideration of all the above factors.

Although it is not well known, many EU member states have similar rules, such as the right to quote, etc. Setting aside
the fact that Article 11 would kill the right to quote, we would definitely need something similar in the EU.

These should be legitimate reasons to file a &ldquo;counter notifications&rdquo; against an upload filter hit.

## Will it happen?

This train of thought all depends on the establishment of an EU copyright registry, which is in and of itself a task
that could take years, so I won't get up my hopes. This proposal would also need to be translated into legalese, which
I'm not qualified to do either. But it's an interesting thought experiment.

From a purely technical standpoint, this is something I could probably stand behind and also vouch for from a technical
perspective. The current proposal... not so much.