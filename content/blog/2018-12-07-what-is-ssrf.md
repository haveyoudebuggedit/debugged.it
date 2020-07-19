---
categories: blog
date: "2018-12-07T00:00:00Z"
publishDate: "2018-12-07T00:00:00Z"
excerpt: Cross-site Request Forgery or CSRF is a well known security vulnerability.
  But what is SSRF?
fbimage: /assets/img/what-is-ssrf.png
googleimage: /assets/img/what-is-ssrf.png
preview: /assets/img/what-is-ssrf.jpg
sharing:
  discord: '@everyone new article about security: What is Server Side Request Forgery?'
  facebook: '#SSRF is a sneaky #security vulnerability that may cause you big headaches.
    Let''s talk about it!'
  linkedin: SSRF is a sneaky security vulnerability that may cause you big headaches.
    Let's talk about it!
  patreon: SSRF is a sneaky security vulnerability that may cause you big headaches.
    Let's talk about it!
  twitter: '#SSRF is a sneaky #security vulnerability that may cause you big headaches.
    Let''s talk about it!'
tags:
- Development
- DevOps
- Security
title: 'Security: What is Server Side Request Forgery?'
twitter_card: summary_large_image
twitterimage: /assets/img/what-is-ssrf.png
---

[CSRF](/blog/what-is-ssrf) is, while still prevalent, no longer an unknown type of security vulnerability. It's brother,
SSRF, however, is. Most developers, or even DevOps engineers don't know about it. So how does it work?

## SSRF explained

SSRF is, at it's core, very simple. Let's say you have a web application that needs to download some data from external
sources. So we trick this web application to, instead of accessing the external site, download the data from an internal
server that is publicly not accessible.

Sounds unimpressive, right? However, keep in mind that more and more database, like redis, etcd, or ElasticSearch can
be accessed over HTTP. What if I trick the application to download some data from etcd? Maybe your AWS API keys are
stored in there and I can just grab them?

<figure>{% plantuml %}
{% include skin.iuml %}
hide footbox
actor Attacker
database "Internal service"
control "Web application"
group Direct attack
   |||
   Attacker ->x "Internal service": A direct attack does not work due to a firewall
   ||| 
end
...
group SSRF
   Attacker -> "Web application": Tricky request to the application
   activate "Web application"
   "Web application" -> "Internal service": Request to an internal service
   activate "Internal service"
   "Internal service" --> "Web application"
   deactivate "Internal service"
   "Web application" --> Attacker: Data from internal service is returned to attacker
   deactivate "Web application"
end
{% endplantuml %}<figcaption>Figure: How SSRF works</figcaption></figure>

Or, if I can trick the application to make a PUT or POST request instead of a GET request, I may even be able to change
data.

## Where SSRF happens

Now, you may be asking in what kind of scenarios SSRF can even happen. After all, not many web applications download
data from external sources at the request of the user?

You may think that, but just a few obscure examples include:

- Downloading an RSS feed or other data source from a remote server.
- Displaying snippets or previews of links.
- Partial code execution vulnerabilities in the application.

Even if your application *initially* doesn't download data from external sources, later on there may be a business
requirement and you may not think of it.

Additionally, even a developer could be tricked into opening a link that, under certain circumstances, could load data
from the internal service.

## A practical example

Let's look at a practical example. Many social networks nowadays integrate [OpenGraph](http://ogp.me/) as a method to
render previews such as this:

<figure><img src="/assets/img/ssrf-facebook-preview.png" alt="" /><figcaption>Image: How Facebook renders previews</figcaption></figure>

Now, think about how this works: the user pastes a link and some server side component needs to download the information
from the server. Assuming that your `etcd` database lives on `10.2.0.1`. What is the user pastes this link?

```http://10.2.0.1:2379/v2/keys/amazon-api-key```

If your etcd doesn't use authentication, your application will happily download the super secret API key from the etcd
server and display it to the user.

Or, let's say you are running on [AWS](https://aws.amazon.com). What if the user enters the address
`http://169.254.169.254/latest/user-data/`? (This is the
[AWS metadata API](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) that lets you access
various things about the instance the requesting code is running on.) Or, even more devious, the user sets up a domain
that points to this IP address? This will let the user access your user data, which may contain sensitive data.

## Defense against SSRF

I've talked about layered security before, so my recommendations follow that principle. You will want to have more than
method of defense.

First of all, all internal services, no matter if they are behind a firewall or not, should be put behind some sort
of authentication. Sometimes that will be usernames and passwords, sometimes it will be TLS client certificates. (TLS
certificates will also enable encryption of the traffic.) If you use authentication, that will prevent clients that
don't have access to the authentication data to connect the service.

Second, you should (of course) set up an internal firewall between your application servers and internal services. If
you are using Kubernetes, you should set up a network policy.

And last, your application should vet the URL. At the very least it should check what IP address the URL resolves to.

## Conclusion

SSRF is a tricky vulnerability because it abuses a trusted component of your system: the application code. It's hard to
think of our own components as a potential threat.
