---
slug: the-big-exim-tutorial
guid: https://pasztor.at/blog/the-big-exim-tutorial/
authors:
- janos
categories:
- Technology
date: "2010-03-22T00:00:00Z"
publishDate: "2010-03-22T00:00:00Z"
summary: In September 2009 I created the big Exim tutorial consisting of 5 parts on the Hungarian Unix Portal. In January 2010, I transfered it to my Hungarian site. Now I’m translating it to English.
tags:
- DevOps
- Exim
title: The Big Exim Tutorial
---

## The exim.conf structure

### Exim basics

Most software comes with some sort of default configuration. With Exim, things are a little different. It’s advisable to start with the slates clean, a shiny new emty config file. As you might have guessed, there is a reason for that: Exim is not only an MTA, but contains a whole programming language, a Perl interpreter and much, much more to become what it is today: the most customizable SMTP server known. You are the one to tell the server where, when and how to route and deliver mails, what to transform, which rows to delete or to add. You can count mails, execute MySQL queries of search in an LDAP database as you like without being bound by table layout or schema definitions. You can execute external programs or even read information from a Unix socket. There is a huge amount of information in the [Exim specs](http://exim.org/exim-html-current/doc/html/spec_html/index.html) and it can’t be too close if you’re stuck on something. There is almost no limit to what Exim can do. (It won’t make you coffee, though.)

**Please note:** if you wish to replace an existing Exim instance, you can always start a second one. Just make sure, your PID file, log and spool directories are different.

### Exim config file structure

Enough talk, let’s do something useful. Open an empty editor of your liking and insert the following skelleton:
```
begin acl

begin routers

begin transports

begin retry

begin rewrite

begin authenticators
```

Lots of headings. Lets slice it down:

### Main configuration

Before the ACL’s we have the option to adjust a load of generic parameters. To set the primary hostname of the Exim instance add the following line:
```
primary_hostname = mail.acmecorp.net
```

There are lots and lots of other parameters. Some I will talk about, others you may find in the Exim specs. Just a remark: don’t think Exim can only use this name in HELO. The SMTP transport is as configurable in Exim as anything else.

If you are working on a more complex setup, you can also define macros here which are expanded when the config file is read. For example:
```
USER_QUERY = SELECT CONCAT(account, '@', domain) FROM accounts LEFT JOIN domains ON accounts.domain_id=domains.id
```

### ACL’s

ACL’s (access control lists) are, as the name adequately describes, access control rules. These are used to accept, defer or deny incoming e-mail. It’s important to put the desired ACL rules into the appropriate section, each corresponding to a state in the SMTP transfer process. (You might want to read a little bit about SMTP if you don’t know these.) Before begin acl you must assign ACL aliases like this:
```
acl_smtp_rcpt = acl_check_rcpt
```

We did nothing else here but tell the MTA to run the named check list on each RCPT TO command. Now we can insert the named ACL:
```
begin acl
    acl_check_rcpt:
        accept local_parts = postmaster
```

According to this, we will accept all mail to postmaster@*. Of course this would be a kinda dumb thing to do, it’s just an example. :)

### Routers

Routers are responsible for routing mail (duh). These rules are processed in the order they appear.

A word of warning: **you must make decisions here**, not in the transports! If you need to deliver mail to a different folder for example, you must use separate transports!

Let’s look at the standard delivery router:
```
local_deliver:
    driver = accept
    domains = +local_domains
    local_part_suffix = "-*"
    local_part_suffix_optional
    transport = local_user
```

As you might deduce, we accept all mail to local_domains and pass it on to the transport named local_user.

### Transports

Transports are responsible for mail delivery. They do nothing but configure a driver to process the mail from the router (yes, every type of transport is a driver). Take look at the previously mentioned local_user transport:
```
local_user:
    driver = appendfile
    directory = "/home/${local_part}/.Maildir"
    maildir_format
    delivery_date_add
    envelope_to_add
    return_path_add
    mode = 0660
```

We just delivered a mail to a maildir in a local users’ folder. This could be the end of the line, but there are some important options to configure.

### Retry rules

With Exim you can specify different retry rules for different targets. However, retries do not run at a cronometric intervals but at predefined queue run times. Let’s look at a simple example:
```
* * F,2h,15m; G,16h,1h,1.5; F,4d,6h
```

This rule states that for all destination addresses and all errors retries should occur at every 15 minutes for the first 2 hours, then at a geometric interval until the 16′th hour starting with a 1 hour interval and a factor of 1.5\. Finally try every 6 hours for 4 days. (Math students at the advantage.)

### Rewrite rules

This section is designed to rewrite e-mail addresses for processing. Personally I think that there are better methods for rewriting.

### Autentication rules

This section specifies the authentication methods you wish to support (PLAIN, LOGIN, etc.) Here is a simple example of a PLAIN authenticator:
```
fixed_plain:
    driver = plaintext
    public_name = PLAIN
    server_prompts = :
    server_condition = ${if and {{\
            eq{$auth2}{username}\
        }{\
            eq{$auth3}{mysecret}\
    }}}
    server_set_id = $auth2
```

## A simple MX

As a first practice we’ll put together a simple mail receiver. No fancy magic, just receiving mail and delivering it to the maildir. To ease the configuration, we’ll set up a virtual user scheme because it’s easier and more common these days. Of course having maildirs in the users’ homedirs isn’t much more difficult either but it’s kinda less flexible.

### Main config

As previously said, global config first.
```
local_interfaces=127.0.0.1 : 127.0.0.2
```

We just specified two IP addresses on which Exim should listen. Two remarks here:

*   What you see here is a list. Most options written in plural form are lists. Values are separated by a colon. If you wish to write a colon in the value itself, you must duplicate it (such as with IPv6 addresses).
*   If you are a newbie in Linux, I have a recommendation for you: you should only run services on the exact IP address you design them to. A generic 0.0.0.0 address is not generally advisable. You can check `netstat -nlp` for misconfigured services possibly posing a security risk. (On the other hand if you are a newbie to Linux, should you really be configuring Exim?)

Let’s continue:
```
daemon_smtp_ports = 25
```

You might have guessed, this specifies the ports to listen on. As before, this is a list. If you wish to use sepecific IP-port pairs, you can use local_interfaces and add the port number with an extra dot:
```
local_interfaces=127.0.0.1.25
```

Ok, now let’s add the MySQL connetion parameters:

```
hide mysql_servers = 127.0.0.1/db/user/pass
```

Here are some further lists. I’m counting on your deductive reasoning skills to figure the meanings out. As you can see, options may also be provided as a (MySQL) lookup.

```
domainlist relay_to_domains =
domainlist proxy_domains    =
domainlist local_domains    = acmecorp.org : \
                              ${lookup mysql {\
                                  SELECT\
                                      domain\
                                  FROM
                                      domains\
                                  WHERE\
                                      domain='${quote_mysql:$domain}'\
}}
hostlist relay_from_hosts=
```

Now we need the primary host name already discussed previously:
```
primary_hostname=mail.acmecorp.org
```

A very useful feature of our favourite MTA is to hide the true identity:
```
smtp_banner="$primary_hostname Microsoft ESMTP MAIL Service, Version: 5.0.2195.6713 ready at $tod_full"
```

Now, that’s nasty. Speaking of paranoia, let’s trim the inserted received header as well:
```
received_header_text = Received: ${if def:sender_rcvhost {from $sender_rcvhost\n\t}{${if def:sender_ident {from ${quote_local_part:$sender_ident} }}${if def:sender_helo_name {(helo=$sender_helo_name)\n\t}}}}by $primary_hostname ${if def:received_protocol {with $received_protocol}} ${if def:tls_cipher {($tls_cipher)\n\t}}${if def:sender_address {(envelope-from < $sender_address>)\n\t}}id $message_exim_id${if def:received_for {\n\tfor $received_for}}
```

What you see here is a fairly complex lookup, one of the main features of Exim.

To filter viruses, spammers and other wrongdoers, we’ll need a reverse DNS lookup, so let’s set it for all connecting addresses.
```
host_lookup=*
```

Nothing else left to do here, continue on to ACL’s:
```
acl_smtp_rcpt = acl_check_rcpt
acl_smtp_data = acl_check_data
```

### <a name="mx-acls" id="mx-acls"></a>ACL’s
```
begin acl
    acl_check_rcpt:
```

The following rules are going to be evaluated after the `RCPT TO` command. Without getting into the very details, let’s talk a little about the SMTP envelope to avoid misunderstandings.

It is a common misconception to check `To` or `From` headers after the `RCPT TO` command since the SMTP envelope only states the current physical recipient of the letter, not the fancy header in the mail body. The difference is subtle but important, just think of `BCC` or mail forwarding. So remember, that `From`, `To`, `CC`, etc. headers can only be checked after the `DATA` command.

Returning to the ACL’s, our goal is to only accept mail which we can deliver locally. If we don’t check this, we may either generate a bounce message on our side or be an open relay which is just plain bad and generates junk. This is not only irritating but easily gets your mailserver on a blacklist.

Now, the ACL runs at connection time so if you reject a message the sending server is responsible for sending a bounce message to the sender (or just ignores the mail anyway).

There are 4 types of return values:

<table class="table">
<tbody>
<tr>
<th>
accept
</th>
<td>
If the conditions are fulfilled, the message is accepted in the current ACL list. If the conditions are not fulfilled, the next rule is processed. If the ACL contains an endpass keyword failure in rules below the keyword results in rejection. For example:
<pre>
accept domains = +local_domains
       endpass
       verify  = recipient
</pre>
Just to practice, please evaluate the ACL above.
</td>
</tr>
<tr>
<th>
deny
</th>
<td>
If conditions are fulfilled, the mail is rejected. Example:
<pre>
deny message = Recipient verification failed
     !verify = recipient
</pre>
</td>
</tr>
<tr>
<th>require</th>
<td>
This rule only continues to the next, if the conditions are fulfilled. For example:
<pre>
require verify = sender
</pre>
</td>
</tr>
<tr>
<th>warn</th>
<td>This rule always continues to the next one. You can append headers to the incoming mail:
<pre>
warn message  = X-blacklisted-at: $dnslist_domain
     dnslists = blackholes.mail-abuse.org : \
                dialup.mail-abuse.org
</pre>
<strong>Warning!</strong> Even if this example contains a blacklist check, you should not do this since it’s too crude of a method.
</td>
</tr>
</tbody>
</table>

Knowing this, our ACL list should look like this:
```
require verify  = sender
require verify  = recipient
accept  domains = +local_domains
        endpass
        verify  = recipient
deny    message = Relaying is not permitted.
```

Wasn’t that hard, was it now? Just to practice, let’s put together a MIME check in the DATA ACL:
```
acl_check_data:
    deny   message   = $found_extension files are not accepted here
           demime    = com:exe:vbs:bat:pif:scr
    deny   message   = Serious MIME defect detected ($demime_reason).
           demime    = *
           condition = ${if >{$demime_errorlevel}{2}{1}{0}}
    accept
```

### Routers

We’ll have a single router to handle local delivery:
```
local_deliver:
    driver            = accept
    condition         = ${lookup mysql {\
                            SELECT\
                                CONCAT(account, '@', domain) \
                            FROM accounts\
                                LEFT JOIN domains ON accounts.domain_id=domains.id\
                            WHERE\
                                accounts.account='${quote_mysql:$local_part}'\
                                AND domains.domain='${quote_mysql:$domain}'}}
    domains           = +local_domains
    local_part_suffix = "-*"
    local_part_suffix_optional
    transport         = local_user
```

As you can see, we use a MySQL lookup as a condition. If the routing fails, the message becomes unroutable.

### Transports

Just as before, we need to set up the local_user transport:
```
local_user:
    driver    = appendfile
    directory = "/home/vmail/${substr_0_1:$domain}/${substr_0_2:$domain}/${domain}/${local_part}/Maildir"
    maildir_format
    delivery_date_add
    envelope_to_add
    return_path_add
    mode      = 0660
```

As you may have noticed, the Exim expansion language is not only suitable for lookups but in this case, creating the substrings. As a complete description of the Exim language would be a document of it’s own, just see the Exim specs for details.

## Spam filtering

An MX is a nice thing but if you don’t protect yourself against spam your mailbox will paint a sad picture.

### ACL rules

First a few ACL rules which filters about 95% percent of the spammers. Since spammers use botnets which live on end users’ computers, they can easily be filtered by their reverse DNS entries.

To enable reverse DNS checking, be sure to enable the following option in your main configuration:
```
host_lookup = *
```

Now insert the following ACL rules:
```
#Host has no reverse
deny message     = Client host rejected: no or inconsistent reverse DNS set.
     log_message = no reverse DNS
     condition   = ${if !def:sender_host_name}
#Host has no reverse
deny message     = Client host rejected: no or inconsistent reverse DNS set.
     log_message = no or inconsistent reverse DNS
     condition   = ${if isip{$sender_host_name} {yes}{no}}
#Host has no reverse
deny message     = Client host rejected: no or inconsistent reverse DNS set.
     log_message = no or inconsistent reverse DNS
     condition   = ${if eq{$sender_host_name}{} {yes}{no}}
#Reverse does not match forward record
deny message     = Client host rejected: reverse DNS does not match forward
     !verify     = reverse_host_lookup
#Blacklist dynamic pools
deny message     = Client host rejected: reverse DNS indicates dynamic IP
     condition   = ${lookup{$sender_host_name}wildlsearch{/etc/exim4/badsenders}{true}{false}}
```

As you can see, the last rule contains a lookup in the /etc/exim4/badsenders file. (wildlsearch is a wildcard search.) I have been using the following ruleset for some time now:
```
^\N^ppp-(.*)\N
^\N^dsl-pool\N
^\N\.(pool|pppoe|adsl|dsl|dialup|broad|cust-adsl|dynamicip|dyn|)\.\N
^\N^(pool|pppoe|adsl|dsl|dialup|broad|cust-adsl|dynamicip|dyn)\.\N
^\N^(pool|pppoe|adsl|dsl|dialup|broad|cust-adsl|dynamicip|dyn)-\N
^\N([0-9]+)-([0-9]+)-([0-9]+)-([0-9]+)\.\N
^\N([0-9]+)\.([0-9]+)\.([0-9]+)\.([0-9]+)\.\N
\N\.ip([0-9]+)\.fastwebnet\.it$\N
*.dip.t-dialin.net
```

Still, I recommend you create your own list.

Now to narrow the gap even more, let’s filter the HELO parameters a little:
```
#IP literal HELO
deny message     = HELO command rejected: HELO is IP only (See RFC2821 4.1.3)
     log_message = HELO ($sender_helo_name) is IP only (See RFC2821 4.1.3)
     condition   = ${if isip{$sender_helo_name}}
#Invalid HELO (no dot)
deny message     = HELO command rejected: HELO is not an FQDN (contains no dot) (See RFC2821 4.1.1.1)
     log_message = HELO ($sender_helo_name) is no FQDN (contains no dot)
     condition   = ${if match{$sender_helo_name}{\N^\[\N}{no}{yes}}
     condition   = ${if match{$sender_helo_name}{\N\.\N}{no}{yes}}
#Invalid HELO (double dot)
deny message     = HELO command rejected: HELO is not an FQDN (contains double dot) (See RFC2821 4.1.1.1)
     log_message = HELO ($sender_helo_name) is no FQDN (contains double dot)
     condition   = ${if match{$sender_helo_name}{\N\.\.\N}{yes}{no}}
#Invalid HELO (ends in dot)
deny message     = HELO command rejected: HELO is not an FQDN (ends in dot) (See RFC2821 4.1.1.1)
     log_message = HELO ($sender_helo_name) is no FQDN (ends in dot)
     condition   = ${if match{$sender_helo_name}{\N\.$\N}}
#Funny fellas using our name
deny message     = HELO command rejected: Impersonating me [$primary_hostname].
     log_message = HELO ($sender_helo_name) impersonating [$primary_hostname]
     condition   = ${if match{$sender_helo_name}{$primary_hostname}{yes}{no}}
#Using our addess as HELO
deny message     = Client host rejected: You are trying to use my address [$interface_address]
     log_message = HELO ($sender_helo_name) uses my address ($interface_address)
     condition   = ${if eq{[$interface_address]}{$sender_helo_name}}
#No HELO at all
deny message     = Client host rejected: no HELO
     log_message = no HELO ($sender_helo_name)
     condition   = ${if !def:sender_helo_name}
```

Be aware, unfortunately no RFC exists which forbids the use of invalid HELO names. since there are a few sysadmins out there ignoring best practices, you cannot check if the HELO name has a valid A record. It’ll reject a LOT of valid mail.

There is one more thing you can do about spam. Since spammer servers just open up a connection and pipeline the whole data into it, delaying your own greeting a little invalidates earlytalker sessions. You should however prepare a whitelist with known earlytalker servers.
```
acl_smtp_connect = acl_check_connect
begin acl
    acl_check_connect:
        accept condition = ${lookup{$sender_host_name}wildlsearch{/etc/exim4/goodsenders}{true}{false}}
        accept delay     = 5s
```

### <a name="spamassassin" id="spamassassin"></a>Spamassassin

We just arrived at our final but not exacly unimportant stage of the spam filtering. We need to engage Spamassassin with Exim. If we have SA support in our Exim (with Debian it’s the -heavy deb package, with Gentoo the corresponding use flag) we have the opportunity to reject spam connection time. Install the spamassassin packages and make sure spamd is running on the correct address. (We’ll return to the spamd configuration later.)

Insert the following row into the main configuration:
```
spamd_address = 127.0.0.1 783
```

Now you need to insert the following DATA ACL:
```
# Insert spam headers
warn message   = X-Spam-Score: $spam_score\n\
                 X-Spam-Score-Int: $spam_score_int\n\
                 X-Spam-Bar: $spam_bar\n
     condition = ${if < {$message_size}{200k}}
     spam      = spamassassin:true
#Reject mail above the score 10.
deny message   = This message is spam.
     spam      = spamassassin:true
     condition = ${if >{$spam_score_int}{${eval:10*10}} {true}{false}}
```

As you can see, there are two variables. $spam_score and $spam_score_int. Unfortunately Exim can’t really handle non-integer numbers. $spam_score_int contains $spam_score multiplied by 10.

Now for this magic to work, you need to configure your spamassassin. Find your distro’s spamassassin configuration directory. Check the options, there are a lot of modules. You really should enable the Bayes module. The autolearn function works well below 1.0 for whitelisting and above 7.0 for blacklisting. If you plan to handle a lot of mails, you should use MySQL for the Bayes database.

### <a name="spam-delivery" id="spam-delivery"></a>Spam delivery

Before we turn our attention to authenticated SMTP, there is a little addon to spam filtering. For some contrived reason people like to get their spam mails into the spam folder. We’ll solve this within Exim.

Unfortunately, it’s not so simple. If you try to do it with simple ACL rules, you may run into an existing spam header set by an other mail system. In this case the header variable (`$header_X-Spam-Score:`) will contain two number separated by a line break. Since Exim can’t parse this as a number, the letter will be defered..

This chicken-and-egg problem can be solved by using the `$acl_m` variables in conjunction with a system filter. First create a DATA ACL:
```
warn set acl_m0 = $spam_score
     set acl_m1 = $spam_report
     set acl_m2 = $spam_score_int
     spam       = spamassassin:true
```

Now add a system filter to the main configuration:
```
system_filter = /etc/exim4/system.filter
```

And edit the file:
```
if first_delivery then
    headers remove X-Spam-Score:X-Spam-Score-Int:X-Spam-Report:X-Spam-Checker-Version:X-Spam-Status:X-Spam-Level
    if $acl_m2 is not "" then
        headers add "X-Spam-Score: $acl_m0"
        headers add "X-Spam-Report: $acl_m1"
        headers add "X-Spam-Score-Int: $acl_m2"
    endif
endif
```

Using you previous knowledge about routers and transports, you should be able to solve the delivery on your own.

## <a name="authenticated-smtp" id="authenticated-smtp"></a>Authenticated SMTP

If you want to send e-mail, you need authentication. Without authentication you’ll have an open relay on your hands and you really don’t want that. Just to practice a little bit, start with an empty configuration file. Your new configuration will only be able to send e-mail, nothing else.

For Exim there is not much difference between authenticated and non-authenticated connections. When users authenticate themselves, the `authenticated` ACL parameter will be set.

As mentioned before, you may specify authenticators after the begin authenticators keywords. The syntax is similar to previous sections:
```
PLAIN:
    driver           = plaintext
    server_set_id    = $2
    server_prompts   = :
    server_condition = ${if eq{\
                           ${lookup mysql{\
                               SELECT\
                                   COUNT(*)\
                               FROM\
                                   users\
                               WHERE\
                                   email="${quote_mysql:$2}"\
                                   AND pass="${quote_mysql:$3}"\
                           }}\
                       }{1} {yes}{no}}
```

Authentication variables will be present in the variables `$1`, `$2` and `$3` depending on which authentication method you use. The exact configuration must be obtained from the Exim specs.

If you know the protocol, you may notice that the username-password query is stated in the server_prompts option. Since a regular human rarely talks raw SMTP, you better believe the docs in this matter. Since the authenticators are now ready, you need to add a little rule to your RCPT ACL:
```
accept authenticated = *
       control       = submission/sender_retain
deny   message       = Authentication failed
```

There you go, authenticated SMTP. A little addon: if you want to insert the original username to the letter, you can use the following remote SMTP configuration:
```
remote_smtp:
    driver         = smtp
    headers_remove = received:user-agent:sender
    headers_add    = Sender: $authenticated_id
```

You may notice, that a receive header gets inserted for your authenticated SMTP session as well. You should remove this header because a lot of spam filters take all received headers into account and will discard your mail for being sent from a dynamic IP otherwise.

The `server_set_id` option specifies the contents of the `$authenticated_id` variable, which may be used in the other parts of the Exim configuration.

The LOGIN authenticator looks like this:
```
LOGIN:
    driver           = plaintext
    server_prompts   = Username:: : Password::
    server_condition = ${if eq{\
                           ${lookup mysql{\
                               SELECT\
                                   COUNT(*)\
                               FROM\
                                   users\
                               WHERE\
                                   email="${quote_mysql:$1}"\
                                   AND pass="${quote_mysql:$2}"\
                           }}\
                       }{1} {yes}{no}}
    server_set_id    = $1
```

## <a name="dkim" id="dkim"></a>DKIM

E-mail sender verification has been a problem for a long time and a lot of technologies have been invented. Some worked better (reverse DNS), others were worse (SPF). Here is a technology, which promises to work well.

Geeks have been thinking about digitally signing e-mail for a long time. End user signing didn’t work because a lot of end user education would have been neccessary. As a replacement technology, DomainKeys Identified Mail has been invented. DKIM signs mail on the mail server.

You need to generate a private and a public key for you server:
```
openssl genrsa -out private.key 1024
openssl rsa -in private.key -out public.key -pubout -outform PEM
```

Remove the header, the footer and the line breaks from the public key. Create the following TXT records in the DNS record of your domain:
```
_domainkey.domain.com. IN TXT "o=~; t=y"
mailserver1._domainkey.domain.com. IN TXT "k=rsa; p=MIGfMA0GCSqGSIb3DQE..."
```

Explaining the first row:

<table>
<tbody>
<tr>
<th>`o=~`</th>
<td>States that only some but not all mail is signed. If you want to change the policy, state `o=–`</td>
</tr>
<tr>
<th>`t=y`</th>
<td>States that the system is in testing mode.</td>
</tr>
</tbody>
</table>

The second record contains the key itself.

To use DKIM, you need at least Exim 4.70\. Look for the transport for e-mail signing and add the following lines:
```
dkim_domain      = domain.com
dkim_selector    = mailserver1
dkim_private_key =
```

With the last line, you have multiple options. Either add the key file, add the key itself or define a lookup. You can also change the domain and the selector using a lookup.

To verify DKIM signatures in your ACL’s use the `$dkim` variables. More information about DKIM support in Exim can be found in the [official specs](http://www.exim.org/exim-html-current/doc/html/spec_html/ch-support_for_dkim_domainkeys_identified_mail.html). Gmail is an excellent tool for testing DKIM since it adds detailed headers explaining the verification.

## Appendix

### SQL schema

Please note: this is only a possible SQL schema. Create the Exim configuration around your existing data infrastructure, not the other way round.
```
CREATE TABLE domains (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    domain    VARCHAR(255) NOT NULL,
    UNIQUE(domain)
) ENGINE=InnoDB;

CREATE TABLE accounts (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    account   VARCHAR(255) NOT NULL,
    password  VARCHAR(255) NOT NULL,
    domain_id INT,
    INDEX(domain_id),
    INDEX(account),
    UNIQUE(account, domain_id),
    FOREIGN KEY (domain_id)
        REFERENCES domains(id)
        ON UPDATE CASCADE ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE VIEW users AS
    SELECT
        CONCAT(account, '@', domain) AS email,
        accounts.password AS pass,
        domains.domain
    FROM
        accounts
        LEFT JOIN
            domains ON domains.id=accounts.domain_id;
```
