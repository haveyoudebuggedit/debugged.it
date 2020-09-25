---
slug: selfhosting
guid: https://pasztor.at/blog/selfhosting/
authors:
- janos
categories:
- Technology
date: "2018-12-24T00:00:00Z"
publishDate: "2018-12-24T00:00:00Z"
summary: Often when a big service provider messes up, there is an inevitable slew of people who tout the advantages of self-hosting everything. But is it really efficient?
images:
- posts/selfhosting.png
preview: posts/selfhosting.jpg
tags:
- DevOps
title: Host everything yourself!
---

It just now happened that apparently [Slack blocked an irani national living in Canada](https://twitter.com/a_h_a/status/1075510422617219077).
([They have now apologized for it.](https://slackhq.com/an-apology-and-an-update)) And, as it usually is, people started
saying that the person in question should use [Mattermost](https://mattermost.com/), a free, self-hosted Slack
alternative, instead.

I was wondering, how meritorious is their claim that *&ldquo;it can be installed in minutes&rdquo;*? Does it truly
describe the effort required to operate a self-hosted alternative? Let's go through the requirements to run a service
like this in detail and see how much effort it really is.

## Hosting

In order to self-host, you obviously need a host. If you already have an infrastructure set up for your non-production
environment, this is easier. If you, however, have to go look for something you are confronted with an array of options
to run your servers. You would probably take something like a [dockerized deployment](https://docs.mattermost.com/install/prod-docker.html).

It does not look too complicated at first, but there are actually a lot of steps involved, such as installing Docker,
launching the container, deploying TLS certificates. Also, you need to make sure your certificates don't expire.

## Backups/Disaster recovery

The next question is, how do you properly do backups. &ldquo;Just copy the files&rdquo; doesn't really do it because 
you need to make sure the backups are actually working and are consistent. Fortunately,
[Mattermost has a disaster recovery documentation](https://docs.mattermost.com/administration/backup.html),
but most self-hosted software doesn't. Nevertheless, you need to make sure your backups are running and are actually
working, which, if we take it seriously, involves disaster recovery drills.

One important aspect of backups, of course, is the location of said backup. If you store it in the same cloud account
as your installation, a compromise of that account may lead to a complete loss of data. So you need to provide an 
independent storage for your backups, preferably in a manner that even when a delete is triggered using the backup 
scripts credentials, it is archived and can be restored.

The second big question about backups is what to backup? Do you backup only the data, or the application configuration
too? If you backup the data, do you have a manual or an automated script how to restore the service after a full crash?
If you backup the full application, how do you know that the backed up state isn't already compromised?

Generally, I would recommend automating the deployment through some sort of configuration management tool and only
backing up the data, but that's obviously a lot more work than just copy-pasting a couple of commands into the shell.

## Monitoring

As this tool will be one of your most critical tool in your organization, you will also want to have some sort of
monitoring so you know if the disk is getting full, or the service has any other issues. There are tools for this, from
Icinga to Prometheus, but again, you need to set it up, and as ironic as it is, your monitoring needs backups and
configuration management too. So this whole thing falls into the category of *that escalated quickly*.

## Updates

Oh yes, nothing invites malware and baddies more than a software that hasn't been updated in years. Your installation
of whatever system you want to run will have to have quite frequent updates. Preferably tested before deploying into
production. In other words, you have to keep a staging environment around, as well as a continuous deployment system. 
After all, who wants to run manual installation steps all the time?

## Conclusion

Self-hosting anything is by no means easy. You need to treat your foreign components just as well as your own code,
in terms of backups, monitoring and updates. That requires effort.

Now the question becomes, how badly will it hurt your company or team if the third party service you are using kicks
you out? Is the risk worth the effort of self-hosting? Will self-hosting actually solve the issue, or will you trade
one third party company for another?
