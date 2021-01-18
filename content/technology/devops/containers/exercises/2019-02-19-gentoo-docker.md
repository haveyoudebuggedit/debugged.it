---
slug: gentoo-docker
guid: https://pasztor.at/blog/gentoo-docker/
authors:
- janos
categories:
- Technology
date: "2019-02-19T00:00:00Z"
publishDate: "2019-02-19T00:00:00Z"
summary: Gentoo compiles everything from source, which sounds it might not be useful
  for Docker. Yet, it can be made to build a sub-100 MB image for PHP...
images:
- posts/gentoo-docker.png
preview: posts/gentoo-docker.jpg
tags:
- DevOps
- Containers
title: Gentoo as a Docker build system?
---

Recently I ran into the need to launch a PHP Docker container. My initial thought was to just simply use the official
PHP images. Quick and simple, right?

That's what it should have been, until I realized I needed the `intl` extension. Long story short, in order to use 
extensions in the official PHP Docker image, you need to collect all the dependencies by hand, such as libicu and what
not, and basically have the Docker image &ldquo;compile&rdquo; the extension from source.

That got me thinking: compiling from source is such a hassle, because you always have to make sure all your dependencies
are all installed, you have all the dev packages, etc. In 2019 this shouldn't really be a problem, especially since
distros like Gentoo Linux have solved this problem over a decade ago.

Sure enough, Gentoo Linux compiles *everything* from source and using it as a production-grade environment is cumbersome
at best, impractical at worst. However, the Gentoo build system is phenomenal. Could we not use Gentoo to build us a
Docker container from scratch?

## How the Gentoo build system works

As I mentioned Gentoo Linux compiles everything from scratch. To do that it has a really (really!) well maintained
set of build instructions called the portage tree. Let's look at the
[`dev-lang/php` package](https://gitweb.gentoo.org/repo/gentoo.git/tree/dev-lang/php). It contains a couple of files
with the ending `.ebuild`, but they area actually nothing more than glorified shell scripts.

These shell scripts run the build process. However, unlike other distros, you do not just have one way to build a 
package. You can control the build behavior using so-called `USE` flags. Do you want to compile the packages on
your system with IPv6 support? Set the `ipv6` use flag. Want to prevent IPv6? Set it to `-ipv6`. Want SSL support? Set
`ssl`. And so on.

In other words, you can compile your system in any fashion you want. So why not use Gentoo in Docker containers
directly?

The answer is: size. The portage tree and the build system are massive. The gentoo/stage3-amd64 docker image is a
whopping 1.7 GB in size, plus the 200 or so MB for the portage tree.

It also makes no sense whatsoever to carry the source code for all the services around in your production environment,
you really just want to have the binaries.

## Building the image

We are standing on the shoulders of giants here. The [Gentoo in Docker](https://github.com/gentoo/gentoo-docker-images)
project has done most of the heavy lifting for us, we simply need to copy-paste the initial code into our `Dockerfile`:

```
# Use the empty image with the portage tree as the first stage
FROM gentoo/portage:latest as portage

# Gentoo stage3 is the second stage, basically an unpacked Gentoo Linux
FROM gentoo/stage3-amd64:latest as gentoo

# Copy the portage tree into the current stage
COPY --from=portage /usr/portage /usr/portage
``` 

This will give us a clean Gentoo image with the portage tree installed using
[multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/).

Now, if we simply use the `emerge` command to install our packages, they will be mixed together with the massive Gentoo
base image. Luckily, we can tell `emerge` to compile the packages into a *different* directory. In other words,
we can create a clean set of files with only the required libraries, without all the Gentoo stuff in it.

To do this we simply need to provide the `ROOT` environment variable like this:

```
ROOT=/destination emerge --quiet php
```

Before we integrate it into our Docker container, let's run it by hand and see what gives:

```
janoszen@janoszen-laptop:~/janoszen/gentoo$ docker run -ti gentoo
977a0f6ec03b / # ROOT=/destination emerge --pretend dev-lang/php

These are the packages that would be merged, in order:

Calculating dependencies... done!
[ebuild  N     ] virtual/libintl-0-r2 to /destination/ ABI_X86="(64) -32 (-x32)" 
[ebuild  N     ] app-arch/bzip2-1.0.6-r10 to /destination/ USE="-static -static-libs" ABI_X86="(64) -32 (-x32)" 
[ebuild  N     ] sys-libs/ncurses-6.1-r2 to /destination/ USE="cxx unicode -ada -debug -doc -gpm -minimal -profile -static-libs -test -threads -tinfo -trace" ABI_X86="(64) -32 (-x32)" 
[ebuild  N     ] app-misc/c_rehash-1.7-r1 to /destination/
...
```

As you can see this output lists all the packages that will be installed. Since the `destination` directory doesn't 
contain anything yet, it will compile all packages, even the base operating system, with one notable example: glibc.
We will talk about that one later.

So, let's put it into our `Dockerfile` and compile PHP!

```
RUN ROOT=/destination emerge --quiet dev-lang/php
```

If you run this command, you will probably sit around for a good half hour until the entire chain of 45 packages
compiles. At the end you, unfortunately, will see a very cryptic error message:

```
!!! Error: invalid target php7.2 for SAPI cli
```

## Solving the eselect issue

The reason we get this error message is the eselect utility. It is a tool in Gentoo that lets you switch between
different PHP versions by switching the `/usr/bin/php symlink`, and it manages other packages as well. Unfortunately it
is not compatible with using the `ROOT` option at the time of writing, which leads to this error.

However, we don't need that in our Docker container, we can set it by hand. So let's replace the eselect utility with
a script that does nothing:

```bash
#!/bin/bash
``` 

Then we copy it into a place where it won't bother anyone:

```
COPY eselect /usr/local/sbin/eselect
```

This will take care of our eselect issue.

## Compiling glibc

I mentioned previously that glibc is not compiled as a dependency, which will cause all our programs to fail. We *could*
simply install it by running `emerge glibc`, but compiling glibc requires the `CAP_SYS_PTRACE` capability for some 
reason, which is not available in a Docker container during build. So we need a trick.

The trick is that we will simply copy the glibc files from the build container. First we query the list of files
in the glibc package:

```
equery -C files glibc
```

This will give us a list of all files in the glibc package. We then run it through rsync to copy the files over
to our `destination` folder:

```
RUN for i in $(equery -C files glibc); do \
      if [ -f $i ]; then \
        mkdir -p $(dirname /destination$i) && \
        rsync -avz $i /destination$i \
      fi \
    done
```

This will effectively copy all glibc files over to the destination for our programs to use.

## Setting use flags

Now comes the fun part: let's change how PHP is compiled. To do that we simply have to set the `USE` environment
variable in the beginning of our Dockerfile. Let's say we don't want IPv6 support:

```
ENV USE="-ipv6"
```

Next, we need to recompile the host operating system so the libraries installed match the new use flags:

```
emerge --update --changed-use --deep --quiet @world
```

This will essentially recompile the host system, which is sometimes used in the build process. This command should be
added to the `Dockerfile` as the first thing after the `ENV USE` line.

## Making it small

Gentoo isn't built with the expectation that you will want a tiny operating system. Or even better, don't even include
the operating system. After all, in a Docker container we don't need all kinds of shell utilities.

Therefore, we have to hack around a little to get Gentoo to remove all the core utilities. This can be achieved by
forcing the removal of the core packages. The command to do this is as follows:

```
RUN ROOT=/destination emerge --quiet -C \
      app-admin/select \
      app-admin/metalog \
      app-eselect/eselect-php \
      mail-mta/nullmailer \
      sys-apps/coreutils \
      sys-apps/file \
      sys-apps/sed \
      sys-apps/shadow \
      sys-libs/ncurses
```

The list of packages you need to remove may vary. You can check the list of packages installed like this:

```
ls -la /destination/var/db/pkg/*
```

Once you have the list of packages, you can make an educated guess which packages will be needed for running the target
application, in this case PHP.

**Note** that you are actively breaking dependencies here by applying the `-C` flag! This is only recommended if you 
want to go for a really small image, and you don't need that stuff anyway. This should always be the last step in your
build process!

It is also helpful to set the `PYTHON_TARGETS` so the Python dependencies are not installed *before* PHP is compiled:

```
ENV PYTHON_TARGETS=""
```

Finally, it may be worthwhile removing manuals, source code, etc:

```
RUN rm -rf \
        /destination/usr/bin \
        /destination/usr/share/doc \
        /destination/usr/share/gtk-doc \
        /destination/usr/share/eselect \
        /destination/usr/share/info \
        /destination/usr/share/man \
        /destination/var/db/pkg \
        /destination/usr/lib64/php7.2/include \
        /destination/usr/lib64/php7.2/lib/build \
        /destination/usr/share/aclocal \
        /destination/usr/share/gettext \
        /destination/usr/include \
        /destination/var/lib/gentoo \
        /destination/var/lib/portage \
        /destination/var/cache/edb
```

## Making it runnable

Up until now we have only created a runnable set of libraries in our `/destination/` folder. How do we make this into
a compact little Docker container?

The answer is, again, multi-stage builds. Take the `/destination/` folder and make it our container:

```
# Start from an empty image
FROM scratch

# Copy the destination files from the previous stage
COPY --from=base /destination /
```

Finally, we can define an entry point for our container:

```
# Run this command at startup
ENTRYPOINT ["/usr/lib64/php7.2/bin/php-fpm", "-F", "-c", "/etc/php/fpm-php7.2/php.ini", "-y", "/etc/php/fpm-php7.2/php-fpm.conf"]
```

**This will create a sub-100 MB Docker image with PHP in it.**

## Where is the source code?

Cool! So where's the source code?

The sad answer is, there is none. At least not public. This is very very experimental and I don't want to risk someone
grabbing the Dockerfile and using it. If you want to play around with it, feel free to ask on my
[Discord channel](/discord), I will happily give it to you.

Other than that, this guide should walk you around the pitfalls and get you to a working setup, but you will have to
fill in a couple of blanks. If you manage to do that, I'm pretty sure that you will have the skills to not shoot
yourself in the foot with this.
