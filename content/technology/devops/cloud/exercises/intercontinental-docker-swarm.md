---
type: exercise
slug: intercontinental-docker-swarm
guid: https://pasztor.at/blog/intercontinental-docker-swarm/
authors:
- janos
categories:
- Technology
date: "2017-10-22T00:00:00Z"
publishDate: "2017-10-22T00:00:00Z"
summary: 'Docker is the new hotness. Swarm is an even newer, even hotter thing. The question is: will it blend? Can it run spanning multiple continents?'
images:
- posts/intercontinental-docker-swarm.png
preview: posts/intercontinental-docker-swarm.png
tags:
- Containers
- DevOps
title: Intercontinental Docker Swarm
---

## Preparing AWS

To prepare AWS for this task, we will add 3 EC2 instances in different regions and set up the security groups so that
traffic can flow freely betweek the 3 nodes. Most prominently we will need these ports:

- 2377
- 4789
- 7946

In addition we will need to let through all proto 50 traffic for encrypted overlay networks.

## Try 1: everyone's a manager

OK, so for our first try, let's spin up 3 EC2 instances on AWS and let's get cooking. We'll use the latest Docker CE
on Ubuntu, and we'll start by creating a manager on the eu-central-1 instance:

```
root@eu-central-1:/home/janoszen# docker swarm init
Swarm initialized: current node (dsmvxhw71u6oqpiqfcer1nm6m) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3qv0eic4mpley7779i8hh5qeazc2hke9g829v4afr6zjus56yd-ddppyajfozgtl6r17bontitje 10.0.15.117:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Then we'll grab the manager token on said node:

```
root@eu-central-1:/home/janoszen# docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3qv0eic4mpley7779i8hh5qeazc2hke9g829v4afr6zjus56yd-eq1sytwnf1poruci3ldlxr0my 10.0.15.117:2377

```

And finally, join in the other two nodes:

```
root@us-west-1:/home/janoszen# docker swarm join \
    --token SWMTKN-1-3qv0eic4mpley7779i8hh5qeazc2hke9g829v4afr6zjus56yd-eq1sytwnf1poruci3ldlxr0my \
    eu-central-1.glb.techblog.cloud:2377
This node joined a swarm as a manager.
```

However, after all 3 nodes are in, we have a problem:

```
root@eu-central-1:/home/janoszen# docker node ls
Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader.
It's possible that too few managers are online. Make sure more than half of the managers are online.
```

Ouch! All right, on to try #2.

## Try #2: initialize nodes as workers

This time around we will initialize the nodes as workers instead of managers, so they effectively don't take
part in the raft consensus. This works nicely:

```
root@eu-central-1:/home/janoszen# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
t3vaz68rnvhz1q7hrpgmavp4q *   eu-central-1        Ready               Active              Leader
odcfosdb0bvhfupmi1v3g9vv2     us-east-1           Ready               Active              
fhse6senppory54auepvx96ak     us-west-1           Ready               Active  
```

Let's promote the other two nodes:

```
root@eu-central-1:/home/janoszen# docker node promote us-east-1
Node us-east-1 promoted to a manager in the swarm.
root@eu-central-1:/home/janoszen# docker node promote us-west-1
Node us-west-1 promoted to a manager in the swarm.
```

Cool! Let's check the swarm status on the other nodes:

```
root@us-east-1:/home/janoszen# docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster
state. Please run this command on a manager node or promote the current node to a manager.
```

What the... it seems the node was promoted on the original manager, but failed to join the consensus:

```
root@eu-central-1:/home/janoszen# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
t3vaz68rnvhz1q7hrpgmavp4q *   eu-central-1        Ready               Active              Leader
odcfosdb0bvhfupmi1v3g9vv2     us-east-1           Ready               Active              
fhse6senppory54auepvx96ak     us-west-1           Ready               Active              
```

As you can see the other two node are missing the `reachable` tag. So this ain't working. What's worse, if we now reboot
the node, it won't rejoin the cluster.

## Try 3: fix the swarm advertise address

At this point I realized that I was being a dumbass because the docker swarm was trying to connect the internal AWS
IP addresses, which could not have worked. So let's fix that:

```
root@eu-central-1:/home/janoszen# docker swarm init --advertise-addr $(dig +noall +answer eu-central-1.glb.techblog.cloud | awk ' { print $5 } ')
Swarm initialized: current node (zhk7fz6rn9oz6uiig95c2jk3f) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-33ex0erstwei7jdpl0t9wi74mjiarqjorot3gubct6gpu5s87w-50469ktocz7op6uka8u5xj3kr 18.194.241.96:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

And join with proper addresses too:

```
root@us-east-1:/home/janoszen# docker swarm join --token SWMTKN-1-33ex0erstwei7jdpl0t9wi74mjiarqjorot3gubct6gpu5s87w-50469ktocz7op6uka8u5xj3kr --advertise-addr $(dig +noall +answer us-east-1.glb.techblog.cloud | awk ' { print $5 } ') eu-central-1.glb.techblog.cloud:2377
This node joined a swarm as a worker.
```

Now let's promote the nodes:

```
root@eu-central-1:/home/janoszen# docker node promote us-east-1
Node us-east-1 promoted to a manager in the swarm.
root@eu-central-1:/home/janoszen# docker node promote us-west-1
Node us-west-1 promoted to a manager in the swarm.
```

And what do you know? It worked!

```
root@eu-central-1:/home/janoszen# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
zhk7fz6rn9oz6uiig95c2jk3f *   eu-central-1        Ready               Active              Leader
rz9mixmz2bw50d0cppayy7ind     us-east-1           Ready               Active              Reachable
tw55u6zokn6apzq6hi46yi7t8     us-west-1           Ready               Active              Reachable
```

{{% tip %}}
**Hint:** Always join your swarm managers as workers first. If the join fails, it won't completely bork your cluster.
{{% /tip %}}

## Adding an overlay network

All right, next up we want one of those sexy overlay networks to run across our intercontinental swarm. Needless to say,
we want it encrypted. So here's how we do it:

```
root@us-east-1:/home/janoszen# docker network create --opt encrypted --driver overlay --attachable internal
hik59iuw2swwawozjg0yofwou
```

And now on to the meat of the matter. Let's start a service.

```
root@us-east-1:/home/janoszen# docker service create --replicas 3 --name php --network internal opsbears/php-fpm
z2xj6a2ztp5xs0pemmp8mbnw0
Since --detach=false was not specified, tasks will be created in the background.
In a future release, --detach=false will become the default.
root@us-east-1:/home/janoszen# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                   PORTS
z2xj6a2ztp5x        nginx               replicated          0/3                 opsbears/php-fpm:latest   
```

Hmmmm... it's not starting. Let's debug that:

```
root@us-east-1:/home/janoszen# docker service ps nginx
ID                  NAME              IMAGE                     NODE                DESIRED STATE       CURRENT STATE              ERROR               PORTS
jqnuawewuhjq        php.1             opsbears/php-fpm:latest   eu-central-1        Running             Preparing 28 seconds ago                       
60u420z5suo6        php.2             opsbears/php-fpm:latest   us-east-1           Running             Preparing 16 seconds ago                       
xakuxz3eeabt        php.3             opsbears/php-fpm:latest   us-west-1           Running             Preparing 16 seconds ago   
```

All right, so it is just taking a looooong time to do anything because of the latency. Cool, let's go into two containers and
check connectivity. First we go into one of the containers and get it's IP address. Then we move to another node and
try to connect it:

```
root@eu-central-1:/home/janoszen# docker exec -ti 1d9c416557e9 /bin/bash
root@1d9c416557e9:/# telnet 10.0.0.8 9000
Trying 10.0.0.8...
Connected to 10.0.0.8.
Escape character is '^]'.
```

Holy macrel, it works!

## Doing something useful

All right, now we need to do something useful with the whole shebang. To put the swarm through its paces, we will use
the *highly experimental* Opsbears Galera cluster image. First of all, we create a secret:

```
pwgen 16 1 | docker secrets create GALERA_ROOT_PASSWORD 
```

Create the data directory on all nodes:

```
mkdir -p /srv/mysql
```

And add the initialization file to one node:

```
touch /srv/mysql/galera_initialize
```

Finally start the service:

```
docker service create \
  --name galera \
  --secret src=GALERA_ROOT_PASSWORD,target=/var/run/secrets/mysql_root_password,mode=0400 \
  --env MARIADB_ROOT_PASSWORD_FILE=/var/run/secrets/mysql_root_password \
  --env WSREP_CLUSTER_NAME=default \
  --mount type=bind,src=/srv/mysql,dst=/var/lib/mysql \
  --env WSREP_CLUSTER_NAME=default \
  --env WSREP_CLUSTER_DNSNAME=galera \
  --network internal \
  --mode=global \
  --endpoint-mode dnsrr \
  opsbears/galera
```

As you can see, we are launching this service in `dnsrr` mode. That's because Galera doesn't deal well with the fact
that there's a loadbalancer inbetween when establishing a cluster.

And this is where the first problem creeps in: when querying the DNS for the service name, the initialization
node does not seem to come up:

```
root@086dbb849495:/# dig galera

; <<>> DiG 9.10.3-P4-Ubuntu <<>> galera
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9100
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;galera.				IN	A

;; ANSWER SECTION:
galera.			600	IN	A	10.0.0.2
galera.			600	IN	A	10.0.0.3

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Sun Oct 22 07:33:44 UTC 2017
;; MSG SIZE  rcvd: 68
```

Note, there should be 3 IP addresses in this response.

Furthermore, we have a problem because `dnsrr` will return the entries in a round-robin fashion. This means that the 
PHP node may end up querying the node across the ocean.

## Building an integrated container

This is not good news, so we need to rethink our approach. Docker Swarm, at the time of this writing, does not have
rack awareness in its feature list, meaning that we have to look for something else if we want to solve this problem.

One obvious way would be to put everything, nginx, PHP, and MySQL into one container. So that's what we are going to
do, using [supervisord](http://supervisord.org/). I won't bore you with the details, there are enough examples of using
supervisord in Docker [on my GitHub organization](https://github.com/opsbears), go have a look.

At any rate, we create our service like this:

```
docker service create \
    --name wordpress \
    --secret src=GALERA_ROOT_PASSWORD,target=/var/run/secrets/mysql_root_password,mode=0400 \
    --secret src=WORDPRESS_MYSQL_PASSWORD,target=/var/run/secrets/wordpress_mysql_password,mode=0444 \
    --env WORDPRESS_MYSQL_PASSWORD_FILE=/var/run/secrets/wordpress_mysql_password \
    --env MARIADB_ROOT_PASSWORD_FILE=/var/run/secrets/mysql_root_password \
    --env WSREP_CLUSTER_NAME=wordpress \
    --env WSREP_CLUSTER_DNSNAME=wordpress \
    --mount src=/srv/mysql,target=/var/lib/mysql,type=bind \ 
    --mount src=/srv/log,target=/var/log,type=bind \
    --network backbone \
    --publish target=80,published=80,mode=host \
    --publish target=443,published=443,mode=host \
    --mode=global \
    janoszen/wordpress
```

Note the publish mode. We need to publish it to the host, otherwise swarm may route the request over the
intercontinental connection, which is not what we want. 

## Conclusions

On a technical level Docker Swarm works, even in an intercontinental setting. However, there are myriad problems this
setup doesn't solve. For instance, how do you get your files to be available on all nodes? Does your application 
behave well with Galera? How do you monitor this setup? How do you get your DNS server to balance traffic based on 
GeoIP? These are just a couple of the problems you'll have to figure out for yourself.