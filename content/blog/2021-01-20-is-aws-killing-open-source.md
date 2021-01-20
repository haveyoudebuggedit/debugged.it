---
title: "Is AWS killing open source?"
slug: is-aws-killing-open-source
summary: "Over the past two days fights have broken out all over the Internet discussing Elastic's move to SSPL... is AWS killing open source or is Elastic in the wrong here?"
authors:
- janos
categories:
- Technology
images:
- /posts/is-aws-killing-open-source/social.png
tags:
- Law
- Open Source
date: "2021-01-20T00:00:00Z"
publishDate: "2021-01-20T00:00:00Z"
---

As you may be aware [Elastic has changed the license for Elasticsearch and Kibana to SSPL](https://www.elastic.co/blog/licensing-change) to thwart AWS' taking *their* products and offering a managed service. Elastic follows in the footsteps of [MongoDB](https://www.mongodb.com/press/mongodb-issues-new-server-side-public-license-for-mongodb-community-server) trying to protect their business.

This move has sparked a wide array of opinions ranging from *&ldquo;AWS killed open source&rdquo;* to [&ldquo;Elasticsearch does not belong to Elastic&rdquo;](https://drewdevault.com/2021/01/19/Elasticsearch-does-not-belong-to-Elastic.html).

I'd like to break down this topic into 3 components:

1. [Licensing](#Licensing)
2. [Trademark](#Trademark)
3. [Patents](#Patents)

## Licensing

Any software released nowadays comes with a *license* which directs what you can or cannot do with the software. The only exception is a work-for-hire situation where you get handed over ownership of the software specifically written for you.

*Open source* comes in a variety of flavors ranging from purely exposing the source code for development purposes without any rights to actually run the software (e.g. [Space Engineers](https://github.com/KeenSoftwareHouse/SpaceEngineers)) to the Linux kernel which is licensed under the GPL license.

However, Space Engineers is an outlier. When talking about open source we typically mean software released under licenses that are [published by the Open Source Initiative](https://opensource.org/licenses) or are at least close to them in spirit, for example the [WTFPL license](https://en.wikipedia.org/wiki/WTFPL).

Licenses typically give you the right to *use and distribute the source for free*. Some licenses require you to include a notice (e.g. [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/)), while others require you to distribute the source under the same license (e.g. [GPL](https://choosealicense.com/licenses/gpl-3.0/))

There is one license that stands out among the OSI-approved licenses: the [Affero General Public License (AGPL)](https://choosealicense.com/licenses/agpl-3.0/). If requires a service operator offering a service with a modified AGPL-licensed software to release their modifications to the software.

Let's look at an example called FooDB. Nile Web Services, a cloud computing platform wants to take the AGPL-licensed FooDB and offer a service with it. In order to run FooDB they copy and modify the source code. When they start offering the service they *must* release their modifications to any of their subscribers. They do not, however, have to send it back to the original author. They also don't owe anyone binary builds of their modified software. (Nor does any other open source project for that matter, it's called *open source*, not *free binaries*.)

Nile Web Services also does not have to release any of their tooling they built around FooDB: their build toolchain, their CI system, the web interface they made to order and configure FooDB.

This is where SSPL comes in: it requires anyone offering the service over the network to release the entire platform under SSPL too. In theory SSPL may sound good, but there are severe criticisms.

Let's say you are building a webshop service called Shopificator using FooDB. This service allows users to write advanced FooDB queries in order to grab metrics out of FooDB. Is this considered network access? If yes you must release the entire Shopificator platform under SSPL (ouch!). If you say no then Nile Web Services could simply put a proxy in front of FooDB and circumvent SSPL like that. As far as I know there has been no lawsuit around SSPL to draw out the lines how far network access reaches.

Coming back to Elasticsearch, as [VM (Vicky) Brasseur quite aptly put it: their switch to SSPL creates a business risk](https://anonymoushash.vmbrasseur.com/2021/01/14/elasticsearch-and-kibana-are-now-business-risks) if you are using their products. Their [licensing FAQ](https://www.elastic.co/pricing/faq/licensing) does not change how the license works.

Is Elastic going to sue you? Probably not, Elastic lives off the good will the community. However, managements change and companies are known to try desperate things to keep themselves afloat when they go downhill. In other words, *you are left hoping that Elastic will do well in the future too* or go down with their dignity intact.

## Trademarks

The second aspect of open sourcing is trademarking. The word [&ldquo;Elasticsearch&rdquo;](http://tmsearch.uspto.gov/bin/showfield?f=doc&state=4810:7wx96h.2.2) is trademarked in the US and probably other countries as well.

To quote from the [WIPO website](https://www.wipo.int/trademarks/en/): *&ldquo;A trademark is a sign capable of distinguishing the goods or services of one enterprise from those of other enterprises.&rdquo;* They are established to prevent confusion as to the source of goods.

In other words, you can't take a trademarked FooDB from the example above, modify it and still call it FooDB. The Apache 2.0 license, for example, specifically contains a section for this:

> **6. Trademarks.** This License does not grant permission to use the trade names, trademarks, service marks, or product names of the Licensor, except as required for reasonable and customary use in describing the origin of the Work and reproducing the content of the NOTICE file.

Other open source licenses like MIT do not explicitly grant a trademark license, which is often a point of criticism when it comes to the shorter open source licenses.

When it comes to Elasticsearch they have filed a [lawsuit against Amazon in 2019](https://www.courtlistener.com/recap/gov.uscourts.cand.348946/gov.uscourts.cand.348946.1.0_2.pdf) for **trademark infringement** and **false description/advertising**. The initial complaint is well worth a read, it contains details such as a [tweet by Werner Vogels](https://twitter.com/werner/status/649738362086027265) claiming a partnership between AWS and Elastic for [AESS](https://aws.amazon.com/elasticsearch-service/). To quote from the intial complaint:

> Neither at the time of that tweet nor at any relevant time has there ever been a “partnership” between Elastic and Amazon with respect to AESS.

Amazon offers [a multitude of defenses](https://www.courtlistener.com/recap/gov.uscourts.cand.348946/gov.uscourts.cand.348946.18.0.pdf). Summarizing them:

1. **Failure to state a claim** upon which relief can be granted.
2. **[Nominative fair use](https://en.wikipedia.org/wiki/Nominative_use)**, or the fact that they are only referring to Elasticsearch by its name.
3. **Equitable estoppel:** If Elasticsearch has lead AWS to believe something one way they cannot then claim otherwise in court.
4. **Amazon has received a license** to Elasticsearch. (Apache 2.0, see above)
5. **Laches doctrine:** Elasticsearch should have brought the suit sooner.)
6. **Doctrine of waiver:** Elasticsearch has waived their rights.)
7. **Statute of limitations:** The time within which Elasticsearch could bring the lawsuit has expired.
8. **Failure to mitigate:** Elasticsearch has not done everything in their power to mitigate the issue.
9. **[Unclean hands](https://en.wikipedia.org/wiki/Clean_hands):** Elasticsearch has acted unethically.

It remains to be seen if and in what capacity these hold true. In the end, we may not know at all if AWS and Elastic reach a settlement out of court.

## Patents

Although it does not seem to apply in this case it is worth talking about patents. Software patents do not apply uniformly around the globe, but they do apply in the United States of America which makes them an important point to talk about.

Patents deal with *ideas*, where copyright deals with the ultimate *execution* of an idea. In other words, copyright protects the source code while patents protect the general idea of the inner workings of a software.

As with trademarks, many open source licenses don't explicitly convey a patent license. Some licenses like GPLv3, however, do include provisions to patents.

## CLA's and business models

One thing we haven't discussed yet: business models. When contributing to an open source project the project may or may not have something called a Contributor License Agreement that you need to sign before your change can be merged.

CLA's are a way to mitigate legal liability for the project owner. They require you to sign an agreement that your code will be released under the specified license.

However, CLA's can have another purpose too: more often than not they include the term *&ldquo;sublicense&rdquo*. This allows the project owner to release your code under a new license without your explicit future approval. This is often used as a vehicle to enable dual licensing.

This agreement between you and the project owner can put the project owner in a very special position. They may change the license in a way similar to how MongoDB and Elasticsearch did, or put the project owner in a position where they are the only ones being able to monetize the project.

Is this fair? That's up to how you view open source. On one hand you have the egalitarian idea that open source software is developed by the community so everyone can use the software for their own benefit.

Then you have open source companies that take advantage of open source. MongoDB, Elastic and co have definitely profited from the publicity and easy adoption that comes with a well known license like Apache 2.0. Thousands of people have contributed to these projects under the assumption that the project would stay under this license. Their trust has now been betrayed.
 
However, looking at the [Elasticsearch contributors](https://github.com/elastic/elasticsearch/graphs/contributors) Elastic has undoubtedly poured an incredible amount of money into developing Elasticsearch. Their expectation to make money from their hard work is very understandable.

Amazon is here to stay and they are definitely not beyond [appropriating ideas and projects](https://www.inc.com/sonya-mann/aws-startups-conflict.html). However, open source was never meant to provide a revenue stream for a single company only backed by free labor from the community either.

There is no simple solution to this problem. Projects need developers and not every open source project makes it into a foundation like Apache or CNCF to be developed by a variety of developers. A lot of projects are bootstrapped by a single or a few developers who then want to make their hobby their work and make money off of their hard work. But, we wouldn't have the same willingness to contribute to a proprietary project either.