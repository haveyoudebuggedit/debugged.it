---
slug: interplanetary-filesystem
guid: https://pasztor.at/blog/interplanetary-filesystem/
authors:
- janos
categories:
- Technology
date: "2017-01-11T00:00:00Z"
publishDate: "2017-01-11T00:00:00Z"
summary: Quite by accident, I've stumbled upon a rather interesting technology called
  IPFS. It promises to replace HTTP as a transport protocol for websites and scale
  to interplanetary levels. Even though the claims sound just tiny bit far fetched,
  the technology behind it got me quite excited.
tags:
- DevOps
title: Interplanetary Filesystem
---

Looking at [one of the talks by Juan Benet](https://www.youtube.com/watch?v=HUVmypx9HGI), the principal author of IPFS,
the design goals became quite clear. They wanted to create a protocol that, for all intents and purposes, would replace
HTTP. They wanted to create a system that 1. scales to interplanetary levels 2. preserves the *history* of the web.
Let's see if it works.

{{% tip %}}
**Note:** This article describes the findings based on a very early version of IPFS. By the time you read this
article, IPFS may have changed significantly.
{{% /tip %}}

## A tiny bit of scaling theory

Most web applications today are very centralized. You have one central server that has all the data runs the
application code and then sends you an HTML file that your browser can display. If you run into scaling issues, or you
want to distribute your application across multiple continents, you may end up distributing the code and data across
multiple servers and serving content from multiple locations. You could call this setup *decentralized*, and it comes
with quite a few issues which I've described in [my article about the CAP theorem](/blog/cap-theorem).

But most typical applications don't go as far as to implement a completely distributed system with no core architecture.
Partially because it's really hard, and also because they don't want to hand out all their data and application to
the end user.

Setting the security and licensing concerns aside, what prominent technology do you think about when it comes to
complete decentralization? Could it be BitTorrent? Maybe BitCoin? As it turns out, these technologies have many things
in common. One of the common parts is the use of
[Distributed Hash Tables](https://en.wikipedia.org/wiki/Distributed_hash_table). While I won't go into too much detail
here, you can think of DHT as being a distributed key-value store.

## IPFS data structure

{{% tip %}}
**Note:** this section describes a simplified description of the IPFS filesystem. The actual implementation is
somewhat more complicated than the one outlined in this article.
{{% /tip %}}

Quite unsurprisingly, IPFS uses DHT to distribute information about pieces of data. These pieces of data are stored as
*objects*, consisting of a data block and a links block. The *data* block, as you imagine, contains the data you are
trying to save. The *links* section, on the other hand, has links to other objects. For example:

```json
{
  "Data": "<!DOCTYPE html><html>...",
  "Links": [
    {
      "Name":"robots.txt",
      "Hash":"QmVbqEF2XuC4qrPL27jDZcb3aKHEiFV4zxv68MnsM1rLQy"
    }
  ]
}
```

These objects are described by their *hash*, and object availability is advertised using DHT. As you can see each object
can be linked to multiple other objects with an optional name, etc.

Here comes the funny part: each object can only be stored once. Once you store an object, you can no longer modify it.
So you can safely create a filesystem representation by creating directories as objects with links to other directories
and files. Since you can never change a hash you already stored, there can never be a cycle in the graph, so there won't
be any problems when traversing the directory structure.

Let's look at an example. Here's a directory representation:

```json
{
  "Links": [
    {
      "Name":"index.html",
      "Hash":"QmW8MwJ4CnDMFh5vxf8ARhbUiVcNCRB9tZHmZAnT32Y9L5"
    }
  ]
}
```

So there's one file in this directory. If we add it to IPFS, we'll get this hash:
`QmTnz23Zmx6d8BkHqaszo35JdTjH8mSYYWbqxcLgQ2r7ZH`. From now on, this hash will represent this given state of the
filesystem. What if we added a file?

```json
{
  "Links": [
    {
      "Name":"index.html",
      "Hash":"QmW8MwJ4CnDMFh5vxf8ARhbUiVcNCRB9tZHmZAnT32Y9L5"
    },
    {
      "Name":"robots.txt",
      "Hash":"QmWpcK4hJut7g8m9qatQJ8g392YVa8tnex243W5H4QbxUS"
    }
  ]
}
```

If we add the modified directory object to IPFS, we'll get a different hash:
`QmQBdfMLSLpjNeUdK8knFF7ST4dukrxGSYt1WR36erZY2e` This will represent the modified version of our filesystem, while our
original hash still represents the original version of the filesystem.

Using linking, we could also create a linking to the previous version:

```json
{
  "Links": [
    {
      "Name":"index.html",
      "Hash":"QmW8MwJ4CnDMFh5vxf8ARhbUiVcNCRB9tZHmZAnT32Y9L5"
    },
    {
      "Name":"robots.txt",
      "Hash":"QmWpcK4hJut7g8m9qatQJ8g392YVa8tnex243W5H4QbxUS"
    },
    {
      "Name": ":previous",
      "Hash": "QmQBdfMLSLpjNeUdK8knFF7ST4dukrxGSYt1WR36erZY2e"
    }
  ]
}
```

This way we can traverse the filesystem history back to infinity, as long as the objects represented by the hashes
are reachable.

You might be asking yourself, ok, but with all these versions, how do you know the latest version? After all, you
don't have a pointer to newer versions, only to the older ones.

Luckily enough, IPFS provides you with a companion service called IPNS. Using IPNS, you can create a *mutable* pointer.
Mutable as in you can change where it points. For example:

```
QmbRreCDFvRGYWNBDnCZQHr7uW6SUes2q2xULw2dSkiquH
  -> QmQBdfMLSLpjNeUdK8knFF7ST4dukrxGSYt1WR36erZY2e
```

This IPNS feature is based on public-private key cryptography, so using the private key on your machine you can always
change which has the pointer is pointing at.

## Where IPFS falls short

Fascinated yet? Let me rain on your parade just a tiny bit by posing a question: who pays for the disk space?

Of course, your node has a copy of your data, but what if your computer gets disconnected? As it turns out, IPFS in
its current implementation will not proactively push your objects to other nodes. If nobody else has a copy of your
data and your machine crashes, your data is lost forever. In other words, if you are using IPFS to host your website,
you'll need to make sure your objects are present on at least one server.

Also, the author of IPFS claims that they wanted IPFS to replace HTTP, but HTTP itself has moved away from simply
serving files and IPFS doesn't do much more than that. Sure enough, you can write JavaScript applications that run in
the browser and publish data over IPFS, but then all the data will be *publicly available* and can be mined using DHT
sniffing. Not to mention the fact that writing dynamic web applications over IPFS objects only is incredibly hard since
you forgo all the tools that you have in server-side application development.

## Is it useful?

I'm usually the guy who warns people about jumping on hype trains and using unstable technologies, so I'll do that with
IPFS as well. Don't start implementing IPFS just yet. Even though IPFS comes from an academic field and is well
documented in various papers, and it even has not one but two implementations, it's still a technology that's in the
very early stages. Things may change, and things may break.

That being said, after quite a bit of testing, I think that IPFS could one day become a very useful tool for running a
CDN and distributed data storage backend for your applications. As for replacing HTTP, I don't think that's going to
happen as IPFS is a filesystem, not a complete communications protocol with state.

Finally, I'd like to leave you with [a talk by Vint Cerf](https://www.youtube.com/watch?v=GV0A82TCrf0)
(A.K.A The Architect) about how data is rendered unreadable over time. (Hint: it's not HTTPs fault.)
