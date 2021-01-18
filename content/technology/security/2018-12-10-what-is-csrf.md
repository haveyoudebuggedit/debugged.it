---
slug: what-is-csrf
guid: https://pasztor.at/blog/what-is-csrf/
authors:
- janos
categories:
- Technology
date: "2018-12-10T07:10:00Z"
publishDate: "2018-12-10T07:10:00Z"
summary: Cross-Site Request Forgery is a pretty well known security vulnerability, yet many developers still fail to secure their applications. Let's discuss this!
images:
- posts/what-is-csrf.png
preview: posts/what-is-csrf.jpg
tags:
- Software Development
- Security
title: 'Security: What is Cross-Site Request Forgery?'
---

Let's say you are operating an affiliate system where people can earn a commission for referring customers.
Once a certain amount has been accumulated, users of your affiliate system can request a payout to their bank account.

Now, one day you get a rapidly increasing amount of support requests that money is disappearing from the affiliate
system. After some investigation you find out that users have initiated a payout and the money has been transferred.
When confronted with this, your users deny having done so.

What happened? Have you been hacked? Have your users passwords been stolen? Alarm bells start ringing. Everyone on deck,
all developers feverishly dig through the logs, but nothing crops up. No logins from strange IP addresses, nothing out
of the ordinary. Simply your users initiating transfers out of your system from their machine.

Your developers are stumped. How can this be? The number of complaints is too large to be a coincidence. At a loss
you start poking around, and in one of the forums your users regularly converse you find a strange post. It consists
of nothing but an image, but the image **is broken**. It doesn't load. Coincidentally, the post was made *at the exact
time the strange transfers started happening*.

That's more than strange, so you start to investigate. Right click, inspect. Oddly the image has a very familiar URL:

```
https://yoursite.com/transfers/out?to=attackers-bank-account-number&amount=99999
```

## Say hi to CSRF

This is a true story, I've seen this happen. Not exactly as described, but it happened nonetheless. But what exactly
happened in the background?

Well, what you see is the simplest form of a Cross-Site Request Forgery or CSRF. Every time someone, who was logged
in to your system, visited the forum, the browser would **try to load the URL**. Your site would receive the request
and process it as a legitimate request from the user and send back a webpage. The browser, of course, would not be
able to interpret the webpage as an image, which is why the image looked broken when you looked at it, but that doesn't
matter. The request was sent and your system made the transfer with the users normal login session.

## Defense attempt #1: change it to POST

Your developers, of course, immediately know the answer: let's change the transfer page so it only accepts `POST`
requests, not `GET`. That way when the browser sends a `GET` for the image, nothing would happen.

Yes, it stops this simple attack, but it won't stop all forms of CSRF. Let's say the attacker builds a simple webpage
they lure their victims to. The code is as follows:

```html
<html>
    <body>
        <form
         action="https://yoursite.com/transfers/out"
         method="POST">
            <input
             type="hidden"
             name="to"
             value="attackers-bank-account-number"
            />
            <input
             type="hidden"
             name="amount"
             value="99999"
            />
        </form>
        <script>
          document
            .getElementsByTagName("form")[0]
            .submit();
        </script>
    </body>
</html>
``` 

Very crude, but effective. The little JavaScript snippet will automatically submit the form once the page loads,
initiating a transfer once again. A more sophisticated attack could even load this snippet in an iframe and then
redirect the user to a legitimate page so they don't notice the trickery.

As you can see, simply switching the transport methods will not solve the problem.

## Mounting a proper defense: tokens

Let's find a solution that is actually safe. We do that by requiring the form to include a secret token. This token
is unique to the login session of the user and cannot be read by the attacker.

Our proper form would look something like this:

```html
<form
 action="https://yoursite.com/transfers/out"
 method="POST">
    <input
     type="text"
     name="to"
     value=""
    />
    <input
     type="number"
     name="amount"
     value=""
    />
    <input
     type="hidden"
     name="csrf-token"
     value="super-secret-value"
    />
</form>
```

Now, when the form is submitted, the server side compares the submitted token with the token stored in the users
session on the server side. If it matches, the transfer is let through. If not, the transfer is rejected.

{{% tip %}}
**Note:** It is very important that the token be *unique* to the users login session so that an attacker cannot get a hold of it.
{{% /tip %}}

Alternatively we could also store the CSRF token in a cookie with the users as the attacker will not be able to get
a hold of that without some further vulnerability on your side.

## Single-Page Apps are not immune

You may, of course, think that you are immune to this type of attack if you use a fancy Single-Page App (SPA) and
communicate with the server using APIs.

In some cases, however, I have to disappoint you. If you use cookies for user authentication, you are just as 
vulnerable. The reason is that the browser *automatically* sends the cookies with every request going to your
domain, no matter where it comes from.

## CSRF protection is important

CSRF protection is just as important as preventing [injection-type attacks](/blog/injection-type-vulnerabilities).
Don't neglect it, not even in seemingly innocuous places like a login form.
