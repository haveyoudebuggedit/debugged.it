---
slug: cap-theorem
guid: https://pasztor.at/blog/cap-theorem/
authors:
- janos
categories:
- Technology
date: "2016-10-04T00:00:00Z"
publishDate: "2016-10-04T00:00:00Z"
summary: The CAP theorem is one of the most fundamental principles of distributed
  system design. Yet, it is often misunderstood or outright disregarded.
tags:
- DevOps
title: What is the CAP theorem?
---

When building larger computer systems, the database soon becomes a bottleneck. Scaling out to more than one machine 
is an option, but scaling comes at a cost. And that's where the CAP theorem comes into play: it describes 
how data storage systems behave when run on multiple computers.

Let's assume you have a web platform that uses some form of traditional database. This could be MySQL, PostgreSQL, 
you name it. You have your daily backups configured, everything is running smooth. However, since your platform is 
successful, your server gets more and more load. You hire a consultant to optimize it, you move the SQL server to a 
separate machine, but at some point you can't help but notice that you need to scale out your database to more than 
one server.

Being the cautious person you are, you hopefully start to read up on this topic. After all, if you have two servers, 
which one is “the source of truth”? What if you update the same information on both servers at the same 
time? Granted, these don't seem like big issues at first, but if you don't pay attention to them, they will come back
and haunt you.

## The CAP theorem

With that in mind, please let me introduce today's star of the show: the CAP theorem. As Eric Brewer states in his 
1999 keynote, computer systems can be described with three fundamental properties: consistency, availability and 
partition tolerance. However, any one system can only satisfy two out of three of these properties.

### Consistency

Imagine you are writing data to a database, and you have just finished sending a larger batch of records, but are 
still waiting for the results of your insert. Suddenly the server you were writing to crashes and you are left 
wondering what happened to your data?

“Classic” SQL databases running on one machine usually put great emphasis on making sure your data is either written 
fully, or is not written at all. They also ensure that whoever reads the data in the database, they always get the 
latest state, no old data is returned.

However, that is not necessarily the case when talking about a distributed database setup. Data takes time to 
propagate from one server to the others, and these database engines may not have the facilities to ensure that your 
data is always written in full.

### Availability

This is pretty straightforward. Availability means that you can read and write data to and from any node in your 
cluster, as long as the node you are connected to isn't the one failing.

However, having availability does not mean that the data you are reading is actually up to date, or if the data you 
are writing won't get lost in a subsequent crash. It just says that you can talk to a server and it will talk back to
you.

### Partition tolerance 

Let's face it, networks aren't perfect. Partition tolerance basically means that your database won't blow up if 
your cluster is split in two parts.

### The catch

Easy, right? Well, there's a catch: if you want to run your database on more than one server, **partition tolerance is 
not optional**. Why? Because if there is a network outage between your two (or more) nodes, your database blows up, 
and that would defeat the purpose.

In other words, you're stuck with *having to chose* between consistency and availability.

## Examples

Let's look at a few examples.

[MySQL](https://dev.mysql.com/doc/)? Well, MySQL, by default, isn't built for running in a cluster (except for the
[NDB](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html) or Galera setup). Your average, run-of-the-mill 
MySQL server doesn't satisfy the requirements for partition tolerance, it runs on one server. So no partitions, no 
cluster.
 
What if we run a [master-slave replication](https://dev.mysql.com/doc/refman/5.7/en/replication-howto.html) MySQL
setup? Well, a master-slave replication only allows writes to the master server, while reads can be done from slaves.
Since there are multiple slaves, for *reads only* we have a setup that sacrifices consistency for availability. 
MySQL replication is asynchronous, so we are not guaranteed to have the latest data on our slaves, but, since we have
more than one server, we can always read *something*, which makes the cluster available. For reads only. For writes 
you're fresh out of luck, if the master goes down, you can't write anything.

What about the [Galera cluster](http://galeracluster.com/products/)? Well, the Galera cluster, does *synchronous 
replication*. The difference is that Galera actually runs on top of InnoDB, so you can (mostly) use the things that 
you are already used to. With Galera, for every commit in your database, data is synchronized across the whole 
cluster and you only get a response once the data is successfully written in all nodes of the cluster. Galera also 
enforces quorum on cluster membership, so all the nodes must see each other in order to be proper members of the 
cluster. If a cluster is split in two, the majority of the nodes will form the cluster, the minority will deny any 
reads or writes to preserve consistency. So Galera, on a recommended setup, is consistent and partition tolerant, but
not available.

The [built-in NDB cluster](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html), much like Galera, provides 
*synchronous replication* of data for a single cluster, but for multiple clusters the replication is asynchronous. So
it gets tricky. For a single cluster NDB is consistent and partition tolerant, where as for multiple clusters NDB is
available and partition tolerant. If you want to learn more, read
[this excellent blog post](https://messagepassing.blogspot.co.at/2012/03/cap-theorem-and-mysql-cluster.html). 
One major drawback of NDB is that you have to use the special NDB table type, you can't use InnoDB like you normally
would.

Finally, let's look at [Elasticsearch](https://www.elastic.co/). Elasticsearch is available and partition tolerant, but 
not consistent. If you write something and the node you have written to goes down, your data is gone. Also, you may end 
up with a situation where your node is isolated from the cluster. You can still read from it and write to it, but 
your data may never end up finding its way into the general cluster. Your server may crash and all your writes from 
that one node may be lost. I guess the caveat is that you should have proper monitoring when using Elasticsearch.

Available and partition tolerant systems, like Elasticsearch are also described as **eventually consistent**. If you 
leave a cluster alone long enough, and don't write any more data, they will end up in the same state.

## Where the CAP theorem falls short

The CAP theorem is a very simplistic way of describing data consistency. Let's take
[Cassandra](https://cassandra.apache.org/) for an example. When writing data into Cassandra, you can decide what 
[consistency level](https://docs.datastax.com/en/cassandra/2.1/cassandra/dml/dml_config_consistency_c.html) you want.
You can pick `ANY`, which will return after at least one node has written the data. You can also pick `QUORUM`, where 
you will wait for the majority of the nodes. Or you can pick `ALL`, which waits for all nodes to write the data. And 
there are a lot more options for all your refined tastes.

As you can see, the CAP theorem can't really describe Cassandras behavior. Most Cassandra setups would be available 
and partition tolerant, but from a data safety standpoint Cassandra is much more reliable (on most settings) than 
Elasticsearch, for example. In other words, Cassandra is *consistent enough* for a lot of applications, as stupid as 
that sounds from a theoretical standpoint.

Don't get me wrong, the CAP theorem isn't *wrong* per se, it just doesn't describe every aspect of consistency that 
you may need to know about a distributed system.

## Take away

The take away from this is that you (probably) can't describe a database solution with one acronym. When working with
cluster software, you need to take a long hard look at what the consistency promises are and if they match your 
business requirements.

## Sources

* [CAP Twelve Years Later: How the "Rules" Have Changed](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed)
* [The CAP theorem and MySQL Cluster](https://messagepassing.blogspot.co.at/2012/03/cap-theorem-and-mysql-cluster.html)
* [CAP Theorem and Elasticsearch](https://messagepassing.blogspot.co.at/2012/03/cap-theorem-and-mysql-cluster.html)
