---
slug: setting-up-apache-with-php-fpm
guid: https://pasztor.at/blog/setting-up-apache-with-php-fpm/
authors:
- janos
categories:
- Technology
date: "2013-04-29T00:00:00Z"
publishDate: "2013-04-29T00:00:00Z"
summary: 'Nowadays nginx seems to experience a serious growth in terms of numbers when looking at HTTP server software. Almost all articles regarding PHP-FPM detail the setup with nginx, very few talk about the good old Apache HTTPd. Admittedly, it’s a little harder to set up due to the myriad hacks layered in it’s internal infrastructure. It has one major advantage however: it handles .htaccess files which allows customers to configure their own little corner of the webserver without poking the admin or endangering the server’s stability.'
tags:
- DevOps
- PHP
title: Setting up Apache with PHP-FPM
---

## How PHP-FPM works

To understand the whole setup we must first take a look at PHP-FPM. If you’ve set up PHP with FastCGI under Apache before you may have noticed that Apache runs all the PHP processes and they are executed under a specific user using the _suexec_ binary which runs as root with the <abbr title="SetUserID: on Linux/Unix system marks a binary (application) so regardless who actually runs the binary always runs with the permissions of the user who created the file. The default is to run with the permissions of the executing user.">SUID bit on.</abbr>

Even though PHP could run as a standalone binary and work as a FastCGI server before, it only became widey used with the release of PHP-FPM. In this mode the PHP process runs standalone without the need for a webserver and listens for incoming requests on either a TCP or a Unix socket. Webservers can connect the PHP process and send requests using the FastCGI protocol.

With PHP-FPM however one could define multiple – so-called – pools which could run using different settings and even user ID’s. This made the FPM ideal for multi-tenant hosting.

## Setting up the PHP-FPM

In our examples I’m going to illustrate the process for Ubuntu Linux, the process should however work very similar with other Linux distributions.

First of all if you haven’t already, you need to install the FPM-CGI binary for PHP:

```bash
apt-get install php5-fpm
```

You’ll have a few new files to configure. First of all you need to set up a pool for yourself. To do this take a look at /etc/php5/pool.d/www.conf. The file is full of comments so you should find your way around. The most important option is the `listen` option. This will tell your pool where to listen for connections, by default `listen = 127.0.0.1:9000`. Be sure to check this setting because your Apache will need to connect this port. For the first try I recommend leaving everything as it is and restarting the FPM by running:

```bash
service php5-fpm restart
```

## Setting up Apache

You can set up Apache with two FastCGI modules: mod_fastcgi and [mod_fcgid](http://httpd.apache.org/mod_fcgid/), however **mod_fcgid cannot handle external servers** as far as I know, which makes it pretty useless with PHP-FPM.

mod_fastcgi is the “classic” module from the makers of FastCGI. To make it work with PHP you need to route the request through several detours. In fact it works much like using the suexec binary.

To use mod_fastcgi, you first need to install it on your server:

```bash
apt-get install libapache2-mod-fastcgi
```

Once installed, you need to set up a (non-existent) URL that Apache can route the requests through. This URL must be available under your web root, so if your webroot is `/var/www`, the URL might be `/var/www/cgi-bin/php5.fcgi`. As I said this file doesn’t have to exist at all.

Next you need to point your server to the external server for the aforementioned URL **within your virtualhost configuration** (so within `<VirtualHost ... >`):

```apacheconfig
FastCGIExternalServer /var/www/cgi-bin/php5.fcgi -host 127.0.0.1:9000
```

As you can see, the `cgi-bin/php5.fcgi` URL is _within_ your document root. It is a virtual URL that will capture all requests to that URL and send them to the PHP-FPM. Finally you need to send all files with the .php extension to this URL **within a `<Directory ... >` directive**:

```apacheconfig
AddType application/x-httpd-fastphp5 .php
Action application/x-httpd-fastphp5 /cgi-bin/php5.fcgi
```

This takes all files with the .php extension and defines a MIME type of `application/x-httpd-fastphp5` for them. (You could use any MIME type.) The Action then sends all requests of this MIME type to the virtual URL created above, which is in turn then sent to the external FastCGI server.

And there you go, your PHP should now work! All this fancy magic is needed so simple image files and such don’t get routed to PHP, which would be a performance hog, a security hole AND would break things at times.

If you need to set up multiple FPM pools, you of course need to configure every pool to it’s separate port AND adjust the Apache configuration for each virtual host separately. So a virtual host configuration for a site would look like this:

```apacheconfig
<VirtualHost *:80>
	ServerName fpmtest.localhost

	DocumentRoot /var/www

	FastCGIExternalServer /var/www/cgi-bin/php5.fcgi -host 127.0.0.1:9000

	<Directory /var/www>
		Order allow,deny
		Allow from all

		AddType application/x-httpd-fastphp5 .php
		Action application/x-httpd-fastphp5 /cgi-bin/php5.fcgi
	</Directory>
</VirtualHost>
```

## Setting up multiple FPM pools

When you are setting up _multiple FPM pools_, you will need to assign a separate FastCGI port to each of them. That is port 9000 for the first, port 9001 to the second, etc. You then copy over your default configuration (Ubuntu: `/etc/php5/fpm/pool.d/www.conf`) to a second file (say www2.conf), change the _pool name_ at the top and the _listen directive_.

The next step is to clone your Apache vhost file and change the appropriate directives (ServerName, FastCGIExternalServer, etc). Then restart both the FPM pool and Apache and you should be set.

## Debugging

It’s pretty easy to break this setup. There are however a few common mistakes you should avoid:

### File not found

This is an error by PHP and basically means that it can’t find the file Apache sent it. You should take a look at your configured chroot and document root. If you chrooted, the path needs to be the same for the webserver and the PHP process, so if you request /var/www/index.php, the PHP process should be able to find such a file as well. If you’re in doubt, disable all advanced PHP options or take a look below in the “when all else fails” section.

### When all else fails

If nothing else helps, you need to dig into the FastCGI protocol. Open up tcpdump (or Wireshark for easier viewing) and sniff into the FastCGI traffic. You should then be able to find out what the webserver requested of the FPM.

If that doesn’t help, you need to debug the FPM pool with strace. If you lack the experience, take a look at my description “[Debugging applications with strace](/2012/04/19/debugging-applications-with-strace/)“.

I hope this little description helped to clarify the setup process. If you’re stuck or have more information to add, please use the comments box below.