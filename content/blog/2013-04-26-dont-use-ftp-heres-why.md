---
slug: dont-use-ftp-heres-why
guid: https://pasztor.at/blog/dont-use-ftp-heres-why/
authors:
- janos
categories:
- Technology
date: "2013-04-26T00:00:00Z"
publishDate: "2013-04-26T00:00:00Z"
summary: FTP has been around since the early days of the internet. Even though it’s
  old and cranky a lot of sysadmins, especially those just getting into managing a
  server, still don’t know anything else. FTP is outdated, has a lot of problems and
  sometimes it can be outright dangerous, however it’s wide spread acceptance as an
  easy way for transferring files makes it hard to switch to alternative protocols.
  If you have a choice, don’t use it. I’ll show you why.
tags:
- DevOps
title: Don't use FTP — Here's why
---

FTP is outdated, has a lot of problems and sometimes it can be outright dangerous, however it’s wide spread acceptance as an easy way for transferring files makes it hard to switch to alternative protocols. If you have a choice, don’t use it. I’ll show you why.  

### Background

If you’ve ever looked at the protocol, you’ll realize it uses two connections. A control channel and a data channel. The control channel is opened from the client to the server and transmits only commands, not data. Whenever a client wishes to upload or download data a second connection is opened, called the data channel. By default, if FTP is set to _active mode_, the server opens the data channel back to the client. This very often doesn’t work with clients behind routers, therefore _passive mode_ enables the data channel to be initiated by the client. This data channel is then used to transfer one file. An interesting fact about the data channel, which we will discuss in just a moment, is the fact that it opens the data channel on a _random port_.

### The problems

#### Lack of encryption

FTP in it’s original form doesn’t support encryption, neither for the control channel, nor the data channel. This is a problem because usernames and passwords are transmitted in plain text over the network. Despite the [FTP Security Extensions](http://tools.ietf.org/html/rfc2228) being around since 1997, finding a client software that support FTPS without having to download additional software or libraries is a challenge. If you actually manage to set up FTPS, you’ll most probably run into problems with the more strict firewalls (see below.

#### Firewalling

As mentioned before, FTP uses a multi-channel approach where the data channel is opened on a random port. Unless you intend to disable the firewall completely on your server, you need tricks to make FTP work. The Linux kernel for example includes a module that _sniffs the control channel_ and allows the connection.

This however comes with a catch: if you use SSL (FTPS) on the control-channel, the Kernel can’t sniff the protocol. This means that you have two choices: either enable a complete range of ports FTP can use or use some more sophisticated access control.

Enabling a range of ports is easy, but it’s not without it’s drawbacks. If you don’t have an application firewall in place, you’ll let in/out any and all traffic on those ports, no matter who opened the listening socket or connection. In case of a webhosting environment this can lead to some pretty nasty stuff.

You could also employ a per-application firewall (For example RBAC) which allowed only certain types of binaries to use those ports. This however is rather hard to do, requires quite some maintenance effort and can only be done _if you have your FTP server and your firewall on the same host_.

Your third option of course would be to put some networking filesystem on your hosts and assign a separate IP address to a dedicated FTP server. Needless to say, networking filesystems again come with their own set of problems.

#### Character encoding

Unless you live in some English-speaking country, you’ve almost definitely used some sort of special characters like the German Ä or Ö. The funny thing about FTP is that it doesn’t care about character encoding. Whatever you submit gets used. If your local filesystem runs in ISO-8859-2 and the server runs in UTF-8… tough luck. Your filenames are scrambled and your server-side applications are probably going to have a hard time decoding the invalid UTF-8 characters. Of course you could also teach everyone to use US-only characters, but that happening would be as likely as all software in the world supporting international character sets: not going to happen.

#### Spaces in file names

Here’s a special treat with FTP: the protocol doesn’t use any delimiters around filenames. That means if you have a space at the beginning or at the end of your filename, chances are you can’t even delete it any more. And you can’t delete the folder the file’s in either.

I’ve worked for a webhosting company before and this was by far the most common FTP-related phone call we’d get from clients. After a while we ended up teaching them how to use SCP rather than deleting their files for them because it was easier.

#### Line breaks

Line breaks are an interesting thing. On old Mac’s a line break was a single \r character, on Windows it’s \r\n and on Linux & co it’s \n. To make file transfers compatible, FTP by default transfers in ASCII mode, which means line breaks get translated. Even though most FTP clients nowadays are smart enough not to transfer binary files like images in ASCII mode, there are still people out there having problems with this.

### Alternatives

Of course there are a ton of alternatives, so even listing them wouldn’t fit in a post like this, so I’ll just name two very commonly used FTP alternatives.

#### SCP/SFTP

Unlike FTPS, which is FTP encapsulated in SSL, SFTP and SCP are tunneled through the SSH protocol, which is a single-connection encrypted protocol and used by sysadmins to access server consoles. For your average John Doe user there’s [WinSCP](http://winscp.net/) which works almost exactly as the all-so-famous Total Commander (or you could just get the [SFTP plugin](http://www.ghisler.com/plugins.htm). To enable SCP without giving users shell access, you can use mod_sftp for ProFTPd. In my experience it works very nicely and gives you a sufficiently secure environment so average users can’t wreak havoc on your server.

#### rsync

If you just want to transfer files from site to site, you could just use rsync, which can run on top of SSH. It’s main feature is incremental transfers and there are a ton of other settings what to transfer and what not. Of course it doesn’t exhibit any of the weaknesses mentioned above.

### Summary

If you are running a hosting business, you probably don’t have a choice and have to set up an FTP server. If you can, try to use something like SCP/SFTP because teaching your users to use it is a lot less effort than constantly cleaning up after them.

If you are using a site-to-site transfer, use something more specialized, uniquely suited for a task you are trying to accomplish.
