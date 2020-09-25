---
slug: we-are-at-war
guid: https://pasztor.at/blog/we-are-at-war/
authors:
- janos
categories:
- Technology
date: "2019-05-03T00:00:00Z"
publishDate: "2019-05-03T00:00:00Z"
summary: There's an invisible war being waged in the IT industry. A war to find out which way is the best to produce code.
images:
- posts/we-are-at-war.png
preview: posts/we-are-at-war.jpg
tags:
- Software Development
title: We are at war
---

There's a war going on in our industry. A war to find the best way to produce code. We are fighting if we should use a 
strongly typed language like Java or Ruby, or a weakly typed language like PHP. We are having arguments if static typing 
is the best where the compiler tells you if you made a mistake, or dynamic typing where you will get the type error
runtime.

Proponents of weak typing say that it makes it easier because you don't have to think about types, and you should write
tests to catch errors anyway. Coders of strongly typed languages usually cannot understand this notion. After all,
the type checking catches errors like invalid input and can even be a defense against potential security holes.

The same goes for dynamic versus static typing. Or writing unit tests versus integration tests. Or writing tests at all.

**So which one is correct?**

I, personally, love static typing. It saves me a lot of headaches. As such, you might expect me to say that weak or 
dynamic typing is not good, it won't work. But that's not true.

IT as an industry is growing up. It started out with enthusiasts, and then evolved with small teams. These were the 
individual heroes, cowboy coders, who worked alone and relied on their own personal genius to make sure the code works
as intended.

Nowadays we are writing ever larger code bases, often millions of lines of code. This is larger than a single person 
can maintain, or even keep in active working memory. You need teams, often dozens, hundreds or even thousands of people.
These kinds of systems require different approaches, like strict and static typing, automated tests and QA systems, as
well as splitting up the codebase into microservices.

So is this the pinnacle of software development? Should every company, every developer move into this direction?
Should everyone use statically typed languages, completely cover their code bases with unit tests and write
microservices?

Some people would make us think so. However, think about startups. Often even big companies launch internal startups to
be able to *move faster*, prototype a new idea, a new product. Startups focus on the product and the flexibility to be
able to match changing requirements. Rarely is the initial idea the right one, so startups need to change the whole 
product to follow a new idea.

This **flexibility is more important than stability**. However, when the product matures and makes serious money this
turns around and **stability becomes more important than flexibility**. With maturity comes inflexibility and the
inability to quickly change the whole product. There are tests for each individual part of the application, and for the
application as a whole. Since the system, now grown large, is split into microservices that multiple teams maintain,
compatibility needs to be maintained in their APIs.

This does not mean that these projects cannot bring out new features, or launch new products, but the codebase and
system makes it hard to introduce *radical* changes. Then a new startup comes along, writing code from ground up in
NodeJS (weakly and dynamically typed) and no legacy systems to maintain. With a 10 person team they quickly launch a 
fundamentally new product and become successful over the coming years, at which stage one of the big ones buys them.

They are focussed on one thing, and one thing only. They can *afford* to pick the tools they need and rely on personal
genius rather than the code factory style of work that is going on in big companies.

And here is the crux of the matter: I don't think we all are artisans. I don't think we all are software craftmen
(or women) who write beautiful code. Most of the IT industry is working in code factories. Sure enough, with bean bags,
bright colors, free muesli and coffee, but a factory nonetheless. And **factories require a much different approach**
than following intuition into a creative idea and working in a small team.

Does that mean they can't use microservices? No, not at all. But they are not going to work in a massive software 
development pipeline in a 10 person company. 

There is room, and indeed need for everyone, multiple approaches to the same problem. **It would be really sad if IT
became a thought-monoculture** where only one way of doing things is acceptable. It would be straight up harmful to not
be able to explore a different idea because it would stifle innovation.

So the next time you have an argument about programming languages and call PHP shit or Java oldschool, think about
different perspectives. Maybe the other guy or girl is right from their point of view. And maybe, just maybe, they are 
on to something worth listening to.
