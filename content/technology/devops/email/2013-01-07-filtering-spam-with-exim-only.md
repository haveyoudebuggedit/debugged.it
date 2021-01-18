---
slug: filtering-spam-with-exim-only
guid: https://pasztor.at/blog/filtering-spam-with-exim-only/
authors:
- janos
categories:
- Technology
date: "2013-01-07T00:00:00Z"
publishDate: "2013-01-07T00:00:00Z"
summary: Defense against spam has always been a hassle. Statistical filters only get
  you so far and they consume a LOT of resources. For exactly that reason I like to
  employ basic checking policies before accepting e-mail at all. These policies have
  gotten me pretty far and my false positive rate is pretty low.
tags:
- DevOps
- Exim
title: Filtering spam with Exim only
---

{% raw %}
## Reverse DNS checks

The most basic checks almost everybody employs are the reverse DNS checks. While “normal” DNS (A, AAAA, etc) records tell you the IP address for a domain name (among others), reverse DNS entries (PTR records) tell you what the primary domain name for an IP address is.

Internet Service Providers usually assign large IP blocks to hand out dynamically to their customers and partially don’t bother to set up reverse DNS entries for them, or if they do, they don’t set up “normal” DNS records.

Servers, especially mailservers on the other hand are customary to have their DNS records set up properly. In fact this tendency has become so wide spread, that it is a de-facto law, you can’t send mail without the proper DNS entries.

To check DNS entries for the sending servers in Exim I use the following rules in the RCPT ACL.

```
drop message   = Client Policy Restriction: No (consistent) reverse DNS set.
     condition = ${if !def:sender_host_name}
drop message   = Client Policy Restriction: No (consistent) reverse DNS set.
     condition = ${if isip{$sender_host_name} {yes}{no}}
drop message   = Client Policy Restriction: No (consistent) reverse DNS set.
     condition = ${if eq{$sender_host_name}{} {yes}{no}}
drop message   = Client Policy Restriction: No (consistent) reverse DNS set.
     !verify   = reverse_host_lookup
```

As you can see, I drop the connection right away so the spammers don’t eat up my resources.

## Checking dynamic pools

As discussed before ISP’s assign large IP blocks to customers. Some providers set up a forward-reverse consistent DNS, so the check above doesn’t catch them. An other wide spread tendency is, that ISP’s give names based on the IP address.

To exploit this, I’ve created a file that lists these common patterns. First you need to set up the lookup in the RCPT ACL:

```
drop message   = Client Policy Restriction: Reverse DNS indicates end user IP.
     condition = ${lookup{$sender_host_name}wildlsearch{/etc/exim4/dynamicranges}{true}{false}}
```

Then create the /etc/exim4/dynamicranges file with the following content:

```
^\N.*ppp-(.*)\N
^\Ndsl-pool\N
^\N.*\.(pool|pppoe|adsl|dsl|xdsl|dialup|broad|cust-adsl|dynamicip|dynamicIP|dyn)\..*\N
^\N(pool|pppoe|adsl|dsl|xdsl|dialup|broad|cust-adsl|dynamicip|dynamicIP|dyn)\..*\N
^\N(pool|pppoe|adsl|dsl|xdsl|dialup|broad|cust-adsl|dynamicip|dynamicIP|dyn)-.*\N
^\Nip\-[a-fA-F0-9]+\-.*\N
^\N.*([0-9]+)(\.|-)([0-9]+)(\.|-)([0-9]+).*\N
^\N([0-9]+)-([0-9]+)-([0-9]+)-([0-9]+)\..*\N
^\N([0-9]+)\.([0-9]+)\.([0-9]+)\.([0-9]+)\..*\N
^\N.*\.ip([0-9]+)\.fastwebnet\.it$\N
*.dip.t-dialin.net
*.business.telecomitalia.it
*.ed.shawcable.net
*.codetel.net.do
*.vie.surfer.at
*.pip.digsys.bg
*.dip0.t-ipconnect.de
*.cablenet.net.ar
*.telecom.net.ar
*.anbid.com.br
*.codetel.net.do
*.avc.upei.ca
```

Of course you can extend this list to your liking.

## Filtering based on HELO

Spammers often use botnets to distribute their mails. These machines are mostly infected home PC’s. Fortunately for us they use the machine’s hostname in the HELO command, which is not a fully qualified domain name. Proper mail servers usually use their host name. To detect and block these HELO names, I use the following RCPT ACL rule:

```
deny message   = HELO Policy Restriction: HELO is not an FQDN.
     condition = ${if match{$sender_helo_name}{\N^\[\N}{no}{yes}}
     condition = ${if match{$sender_helo_name}{\N[^.]\N}{no}{yes}}
deny message   = HELO Policy Restriction: HELO is not an FQDN.
     condition = ${if match{$sender_helo_name}{\N^\[\N}{no}{yes}}
     condition = ${if match{$sender_helo_name}{\N\.\N}{no}{yes}}
```

These rules detect missing dots, double dots, etc in the HELO name. There was a time when I also experimented with making a DNS lookup on the HELO names, but unfortunately that didn’t end well. Too many services have internal server names in the HELO which are not resolvable from the outside.

## Filtering Base64 encoded messages

The RFC’s relevant to e-mail define several transport encodings for e-mails, one of them being [Base64](http://en.wikipedia.org/wiki/Base64). Base64 renders any content into an unreadable chunk of data. It is very useful when transmitting binary data, however totally unnecessary when transmitting plain text e-mails.

Modern mail clients send attachments as [multipart messages](http://en.wikipedia.org/wiki/MIME) only encoding the binary parts in Base64\. Spammers on the other hand encode the whole message in the hopes to bypass less sophisticated spam scanners.

What we do is check, if the content type is text/html or text/plain and the transfer encoding is base64\. If both rules apply, we reject the mail. Use them in the DATA ACL of course.

**Warning!** I haven’t seen any mail client, that encodes plain text e-mails into base64, but there may theoretically be some out there. Use this rule with caution.

```
deny message   = Content Policy Restriction: Base64 encoded text messages are not permitted.
     condition = ${if and{ \
                   {eq{$h_Content-Transfer-Encoding:}{base64}} \
                   {match{$h_Content-Type:}{^text/(html|plain)}} \
                 } {true}{false}}
```

## Filtering From and To headers

Most people don’t know, but the From and To headers are only looked at by the sending mail client. Spammers often use this technique to compile a mail once, then distributing the same mail to several thousand mail addresses. I’ve seen the following patterns in practice.

### Undisclosed recipients

The oldest pattern for sending mails to a lot of people is putting all addresses into the BCC header and writing “undisclosed recipients” into the To header. This is now considered an obsolete practice, so filtering such messages is pretty safe.

```
deny message   = Content Policy Restriction: Mails to undisclosed recipients are not permitted.
     condition = ${if eq{$h_To:}{undisclosed-recipients: ;} {true}{false}}
deny message   = Content Policy Restriction: Mails to undisclosed recipients are not permitted.
     condition = ${if eq{$h_To:}{undisclosed-recipients:;} {true}{false}}
```

**Warning!** This rule may lead to rejecting false positives. Decide upon your userbase if you want to enable it or not!

### Mails without a From header

Spammers often like to send mails without From addresses too. The reason is not entirely clear to me, but it can be exploited to filter such messages out since all e-mail clients include their sending address in the header. Add the following rule to the DATA ACL:

```
deny message   = Content Policy Restriction: Messages without From header are not permitted.
     condition = ${if eq{$header_from:}{}}
```

### Mails without a To and CC header

According to the RFC’s all messages must contain a To or CC header. I personally restrict all mails without a To header, but to be compliant you should use the following rule in the DATA ACL:

```
deny message   = Content Policy Restriction: Messages without To and CC headers are not permitted.
     condition = ${if and{{ \
                     eq{$header_to:}{}} \
                   }{ \
                     eq{$header_cc:}{}} \
                 }}
```

**Warning!** This rule may lead to rejecting legitimate e-mails, especially from signup forms, etc. Decide upon your userbase and e-mail patterns, if you wish to use this rule.

### Multiple From addresses

A fairly new tendency in spam is to include multiple addresses in the From header. Mostly these addresses are the same, as the To addresses. This is supposed to confuse spam filters, but again can be exploited to filter such messages in the DATA ACL. For safety reasons we check for three @ signs.

```
deny message   = Content Policy Restriction: Multiple from addresses are not accepted here.
     condition = ${if match{$header_from:}{@.+@.+@}}
```

### Filtering messages with BCC fields

BCC headers are supposed to be filtered out by the first SMTP server the message passes, so if you receive mails for local mailboxes with BCC headers in them, that’s either a spammer or a horribly misconfigured mailserver. Add the following rules to the DATA ACL to filter them.

```
deny message   = Content Policy Restriction: Mails with BCC headers are not permitted.
     condition = ${if !eq{$h_Bcc:}{} {true}{false}}
```

**Warning!** Don’t apply this rule to your local uses, who send messages with authentication, otherwise they won’t be able to send messages with BCC headers!

**Warning!** It seems, that Gmail leaves the BCC headers to the designated recipient in the e-mail body, so this rule would filter such mails.

## Take away thoughts

These rules helped me to radically reduce the amount of spam that actually gets to my statistical filters, thereby decreasing the load on my server. I regularly analyze spams that get through to find new weaknesses in the hopes, that spammers will always come up with new tricks to evade spam filters that can be exploited.

If you have any rules you think are useful, please let me know in the comments below.

{% endraw %}
