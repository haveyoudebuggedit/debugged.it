---
slug: kubernetes-first-major-security-hole
guid: https://pasztor.at/blog/kubernetes-first-major-security-hole/
authors:
- janos
categories:
- Technology
date: "2018-12-05T00:00:00Z"
publishDate: "2018-12-05T00:00:00Z"
summary: Kubernetes first major security hole is out... does this mean Kubernetes is not secure? What can we learn from it?
images:
- posts/kubernetes-first-major-security-hole.png
preview: posts/kubernetes-first-major-security-hole.jpg
tags:
- DevOps
- Containers
title: What can we learn from Kubernetes' first major security hole?
---

Kubernetes is a staple of modern day DevOps. Everybody wants to use it, not using it means you are not part of the
*&ldquo;Kool Kids Korner&rdquo;*. However, just a few days ago a pretty bad security hole tagged
[CVE-2018-1002105](https://access.redhat.com/security/vulnerabilities/3716411) was published. It allows an
unauthenticated user to gain root privileges on the Kubernetes API.

Yes, that is pretty bad, but let's not forget, every software has security holes. By no means is this article intended
throw shade on Kubernetes because of this failure. They fixed and disclosed it properly, end of story.

## The Tutorial Failure

Instead, I'd like to focus on something much different: the failure of tutorials. Every tutorial on Kubernetes I've
seen, even by respected Kubernetes educators like
[Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way), seem to suggest that it is OK to put
your Kubernetes API on a public IP address.

Seasoned security veterans, of course, would never do that, but anyone just learning the topic will take the shown
methods of deployment as a guide how it should be done. Nowhere in any of these tutorials did I encounter a warning that
running a Kubernetes API on a public IP may be a bad idea.

## The Layered Security Principle

I'd like to introduce you to a principle in systems architecture called **layered security**. The basic idea is that if
one security-relevant component fails, there should still be a secondary layer that catches a potential attacker before
harm is done.

Putting the Kubernetes API on a public IP violates this principle because if an attacker can get access to the
Kubernetes API as an admin, they are basically able to run whatever they want on all of your machines. Granted, if the
kubelet is set up properly, they won't be able to run privileged containers, but it's still pretty bad.

## Isolating the Kubernetes API

In my view the Kubernetes API should be isolated from the outside world to ensure multiple layers of security. Deploy
a separate network for Kubernetes where the API runs, and where your kubelets can communicate with your masters. Don't
let anyone from the ourside world in there, and also don't let your production code access any of these APIs. (Unless,
of course, you have a very specific component that needs to talk to Kubernetes.)

Yes, your CI system and your developers need to access these APIs, but that's what VPNs are for. Or if you can't deploy
a VPN, get a fixed IP address and set up a proper firewall to only allow access from your network your developers are on.

Then *test* your security settings using a network scanner like [nmap](https://nmap.org). Can you access your Kubernetes
API from your mobile network? Or can you access it from a container launched *inside* your Kubernetes? If not, then
you're good.

If you can, automate these tests and make sure an alert gets triggered when it happens. Set up network policies in
Kubernetes making sure you can't access the Kubernetes API *or* your cloud providers metadata API.

## Summary

Security is hard and we are failable humans. That means that we need to prepare for the eventuality of failure. We
need to deploy multiple layers of defenses and scan our systems to make sure no oopsies creep in. That, unfortunately,
includes making our systems less convenient and harder to deploy sometimes.
