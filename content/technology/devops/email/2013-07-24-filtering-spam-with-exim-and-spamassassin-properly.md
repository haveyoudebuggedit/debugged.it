---
slug: filtering-spam-with-exim-and-spamassassin-properly
guid: https://pasztor.at/blog/filtering-spam-with-exim-and-spamassassin-properly/
authors:
- janos
categories:
- Technology
date: "2013-07-24T00:00:00Z"
publishDate: "2013-07-24T00:00:00Z"
summary: SpamAssassin is a frequently used companion for Exim. However, most people
  set it up in a synchronous manner – spam is checked directly when the SMTP session
  is opened. While this is certainly a valid technique, it has it’s drawbacks. It
  leaves the server vulnerable to DOS attacks because the spam filtering is a big
  resource hog. Having SpamAssassin headers in the mail from the remote servers is
  also an issue, because the `$h_X-Spam-*` variables will start misbehaving suddenly.
tags:
- DevOps
- Exim
title: Filtering spam with Exim and Spamassassin (properly)
---

For the purpose of this article I am going to assume you are fairly familiar with writing your own Exim configuration and you are also able to set up your SpamAssassin configuration. If you lack either of these abilities, please read up on both topics first.  

## Setting up SpamAssassin

As usual, I will be using Ubuntu as a platform for my tests, but any Linux should work quite similarly. Once you have installed SpamAssassin (`apt-get install spamassassin`) you will find a lot of configuration files in `/etc/spamassassin`. These configuration files regulate how spam filtering is done. **I very strongly recommend reading the Mail::SpamAssassin::Conf man page before continuing!**

So since you read the manual, you now know that there are a lot of options for allowing user-level rules. Therefore SpamAssassin needs a username to act on. This is one of the reasons you need to set up a more complicated setup than just putting the SpamAssassin server in your Exim configuration.

Once you’re done with fine tuning SpamAssassin to your liking, you need to add some headers to filter by. To do this, add the following lines to local.cf:

```
always_add_headers 1
report_safe 0
add_header all Report _REPORT_
add_header spam Flag _YESNOCAPS_
add_header all Status _YESNO_, score=_SCORE_ required=_REQD_ tests=_TESTS_ autolearn=_AUTOLEARN_ version=_VERSION_
add_header all Level _STARS(*)_
add_header all Checker-Version SpamAssassin _VERSION_ (_SUBVERSION_) on _HOSTNAME_
```

This will add the following headers to all your mails passing through SpamAssassin:

```
X-Spam-Report: 
    *  0.0 FREEMAIL_FROM Sender email is commonly abused enduser mail provider
    *      (****[at]gmail.com)
    * -0.7 RCVD_IN_DNSWL_LOW RBL: Sender listed at http://www.dnswl.org/, low
    *      trust
    *      [209.85.214.170 listed in list.dnswl.org]
    * -0.0 SPF_PASS SPF: sender matches SPF record
    *  0.0 HTML_MESSAGE BODY: HTML included in message
    *  0.0 T_DKIM_INVALID DKIM-Signature header exists but is not valid
X-Spam-Status: No, score=-0.7 required=5.0 tests=FREEMAIL_FROM,HTML_MESSAGE,
    RCVD_IN_DNSWL_LOW,SPF_PASS,T_DKIM_INVALID autolearn=ham version=3.3.2
X-Spam-Level: 
X-Spam-Checker-Version: SpamAssassin 3.3.2 (2011-06-06) on
    mail01.in.opsgears.com
```

As you can see, there’s quite a lot of information in there. This will help you to debug any problems you may have with your spam checker. However, for security reasons I recommend removing these headers when sending e-mails to remote servers (e.g. forwarding mails) in the SMTP transports.

## Sending mails to SpamAssassin

So as I mentioned, unless you want to do some frontend baseline filtering for all mails, I recommend completely disabling SpamAssassin checks in Exim itself. This involves removing any SpamAssassin routers and/or the `spamd_address` configuration option.

Instead we will pipe the mail to SpamAssassin using the `spamc` client, then piping it back into Exim, effectively creating a loop:

> Remote server → [via SMTP] Exim → [via pipe] SpamAssassin → [via pipe] Exim → Mailbox

In order to do this, we add an extra router _after any mail forwards, but before any delivery routers_:

```
spamcheck:
    no_verify
    driver         = accept
    condition      = ${if and {\
                         {!eq {$received_protocol}{spam-scanned}}\
                         {<{$message_size}{256k}}\
                     } }
    headers_remove = X-Spam-Flag:X-Spam-Report:X-Spam-Status:X-Spam-Level:X-Spam-Checker-Version
    transport = spam_check
```

This will send all mail that is smaller than 256k AND hasn't yet been checked to SpamAssassin for checking. It will also remove any foreign `X-Spam-*` headers that may have been contained in the mail.

In order to avoid doing double forwards, you should also exclude any routers _before_ the spamcheck router from the looped mails by adding this condition:

```
condition = ${if !eq {$received_protocol}{spam-scanned}}
```

Of course we are also going to need a transport for this:

```
spam_check:
    driver            = pipe
    command           = /usr/sbin/exim -oMr spam-scanned -bS
    use_bsmtp
    transport_filter  = /usr/bin/spamc -u $local_part@$domain
    home_directory    = /tmp
    current_directory = /tmp
```

As you can see, the message is sent back to Exim, using `spamc` as a filter in the process. The e-mail address is passed to SpamAssassin as a username to use when looking up per-user configs.

## Delivering spam mail

Once this is done, spam delivery is quite simple. However, **you need to write your own router and transport**. Do not copy the examples here brainlessly, because chances are they won't work for you.

So you need to duplicate your regular delivery transport and add the following condition to the **first** copy:

```
condition = ${if and {\
                {def:h_X-Spam-Flag:}\
                {eq {$h_X-Spam-Flag:}{YES}}
            }
```

You should also change your `transport` setting to a different name. In my case my router looks like this:

```
mailboxspam:
    no_verify
    condition = ${if and {\
                    {def:h_X-Spam-Flag:}\
                    {eq {$h_X-Spam-Flag:}{YES}}\
                    {eq {${lookup mysql{\
                        SELECT \
                            COUNT(*)\
                        FROM
                            v_accounts\
                        WHERE
                            local_part="${quote_mysql:$local_part}" \
                            AND \
                            domain="${quote_mysql:$domain}"\
                    }{$value}{0}}}{1}}\
                } }
    driver    = accept
    domains   = +local_domains
    transport = spam_delivery
```

As I said, don't copy this!

### Delivering into a Maildir

If you have a standard maildir setup, you need to create a similar transport. Again, don't copy this, write your own.

```
maildir_spam_delivery:
    driver = appendfile
    maildir_format = true
    directory = /your/maildir/path/.SPAM/
    Other transport options here
```

### Delivering with Dovecot

If you are using the Dovecot LDA for delivery, the setup is slightly different. You need to pass the folder to Dovecot using the `-m` parameter like this:

```
spam_delivery:
     driver            = pipe
     message_prefix    =
     message_suffix    =
     log_output
     delivery_date_add
     envelope_to_add
     return_path_add
     user              = dovecot
     group             = dovecot
     command           = /usr/lib/dovecot/dovecot-lda -d $local_part@$domain -a $original_local_part@$original_domain -f $sender_address -m .Junk
     temp_errors       = 64 : 69 : 70: 71 : 72 : 73 : 74 : 75 : 78
```

## Testing the whole setup

Spamassassin has a test string, so if you wish to test your spam delivery, simply send an e-mail with this test string in it:

```
XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X
```

## Frequently Asked Questions

### I added spam filtering, suddenly everything is broken!

You may have made an error with the configuration. Please be sure to test all changes in a development environment first, don't go about editing your production server's configuration.

### Why can't I use SMTP-time (Exim) spam filtering with per-user configuration?

It is due to how SMTP works. When delivering the same mail to several recipients (e.g. CC'd mail), the mail is only delivered once per server, sending multiple `RCPT TO` commands to deliver the mail. The `DATA` ACL is only run once per such a mail, so you can't really filter on a per-user basis, that's why you need the [pipe transport](http://www.exim.org/exim-html-current/doc/html/spec_html/ch-the_pipe_transport.html), because it splits the mail into per-user instances.

### I still have questions open

Exim is a complicated topic and requires a lot of learning. You can't just go about copying someone else's code brainlessly because there is a high probability it simply won't work or even worse, cause a bug you didn't anticipate. You really need to understand what your configuration does. If you need more help with Exim, read my [Big Exim Tutorial](/2010/03/22/the-big-exim-tutorial/).
