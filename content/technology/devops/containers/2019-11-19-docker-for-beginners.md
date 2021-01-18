---
categories:
- Technology
slug: docker-for-beginners
guid: https://pasztor.at/blog/docker-for-beginners/
date: "2019-11-19T00:00:00Z"
publishDate: "2019-11-19T00:00:00Z"
summary: There are a million Docker tutorials out on the Internet, but few explain
  what's actually going on behind the scenes when you issue your first commands. So
  let's take a look!
images:
- posts/docker-for-beginners.png
preview: posts/docker-for-beginners.jpg
tags:
- Containers
- DevOps
title: Getting started with Docker for the Inquisitive Mind
authors:
- janos
---

Ok, so you have decided to start doing something with Docker but you don't even know what is what and how this whole thing works? Maybe you've read a couple of tutorials but you have no clue what you are doing? Then this is the post for you.

## The Docker architecture

Assuming you have followed the official documentation and installed Docker on [Ubuntu](https://docs.docker.com/v17.12/install/linux/docker-ce/ubuntu/) or on [Windows](https://docs.docker.com/v17.12/docker-for-windows/install/), you must question yourself: what is this Docker thing?

Think of it like a very advanced tool to run a virtual machine. Yes, it's not really a VM, but let's stick with this crude analogy for now. What you have installed are two things: the `docker` command (client) and `dockerd` (server). When you open a command line / terminal and type `docker run -ti ubuntu` you actually call the client, which then connects to `dockerd` on your behalf and asks it to run a container (almost a VM) from an *image* called *&ldquo;ubuntu&rdquo;*. This *image* is basically a packaged version of an operating system which is used as a *golden master* to create a copy from. This copy then becomes the *container* itself. (There is a lot more stuff going on behind the scenes, but let's keep it simple for now.)

{{% tip %}}
**Hint for Windows Subsystem for Linux:** If you are trying to access your Docker Desktop on WSL1 (Windows), you will have to enable *Expose daemon on tcp://localhost:2375 without TLS* and run docker as `docker -H tcp://localhost:2375` in order to use the Docker Desktop daemon. WSL2 will fix this by allowing you to run Docker natively within WSL.
{{% /tip %}}

Where does this image come from? Unless you have specified a name that references a private image server (called a *registry*), `dockerd` will try to first look locally in its own storage for the image, and if it is not found, it will contact the [Docker Hub](https://hub.docker.com) to look if the image is available there. If it is, it will download said image and cache it locally.

It is worth keeping in mind that every time you run `docker run`, a *new* container is created from the image, so any modifications you may have made in the old one are not transferred to the new one. This plays heavily into the 
philosophy of Docker: the image build process and the ultimate execution of an image to create a container are two separate steps. The image build process should produce an image that is reusable dozens, or even hundreds of times to 
create an army of nearly identical containers. Images must be *reusable*. Therefore, it is not advisable to go in and edit the contents of a container.

It is also worth noting that the containers you start are running on a *separate, virtual network* where each container has its own unique IP address. Services started in a container are not automatically available from the machine you are running them on. Docker, however, has a clever feature where it will run a *proxy* that forwards connections to the appropriate container, if you ask it to. 

{{% tip %}}
**Further reading:** If you want to know how exactly containers work, I would recommend reading my post titled [Docker 101: Linux Anatomy](/blog/docker-101-anatomy).
{{% /tip %}}

## Running your first container

Enough theory already, let's get to the meat of it. Let's start a container with an Ubuntu Linux in it. Open a terminal and type the following:

```bash
docker run ubuntu
```

{{% warning %}}
**Remember:** If you are on WSL1, you will need to run `docker -H tcp://localhost:2375 run ubuntu` here!
{{% /warning %}}

{{% tip %}}
**Hint:** You can pull in a specific Ubuntu version by typing `docker run ubuntu:16.04` here. More on versions later.
{{% /tip %}}

You will see something like this:

```bat
C:\Users\janoszen>docker run ubuntu
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
7ddbc47eeb70: Pull complete
c1bbdc448b72: Pull complete
8c3b70e39044: Pull complete
45d437916d57: Pull complete
Digest: sha256:6e9f67fa63b0323e9a1e587fd71c561ba48a034504fb804fd26fd8800039835d
Status: Downloaded newer image for ubuntu:latest

C:\users\janoszen>
```

OK, what just happened? First of all, it is clearly visible that Docker just *pulled* the Ubuntu image from the Docker Hub, but then... nothing happened? Ok, let's investigate. Let's take a look at the running containers:

```bat
C:\Users\janoszen>docker ps
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES

C:\Users\janoszen>
```

That's ... unfortunate. No container is currently running, so something clearly went wrong. Let's take a look at all the containers, including the non-running ones:

```bat
C:\Users\janoszen>docker ps -a
CONTAINER ID   IMAGE    COMMAND       CREATED          STATUS                      PORTS   NAMES
78bb7a8bd601   ubuntu   "/bin/bash"   30 seconds ago   Exited (0) 30 seconds ago           kind_almeida

C:\Users\janoszen>
```

OK, so the container was clearly *created*, and it also started, but it immediately stopped. If you look at the 
*command* that is being run, that's `/bin/bash`. Bash is a shell script interpreter, so it expects a shell script to run, unless it is run *interactively*, where you can type. You might assume that's automatic, since you are typing out the Docker commands interactively, but Docker doesn't quite know that, you need to tell it. Let's try this:

```bat 
C:\Users\janoszen>docker run -ti ubuntu
root@f549dcd7db3c:/#
```

Much better! As you can see with the `-ti` added we are now get a Bash prompt where we can type Linux commands (interactive and TTY mode). Feel free to use this to run any Linux commands you want, but keep in mind, this container is running separated from the machine you are running it on and you won't have access to any data that you may have on the host machine. We will fix that later. When you are done, hit `Ctrl+D` to exit the prompt.

If you now run `docker ps -a` (`-a` for all) you will now see that there are two stopped containers:

```bat
C:\Users\janoszen>docker ps -a
CONTAINER ID   IMAGE    COMMAND       CREATED         STATUS                     PORTS   NAMES
f549dcd7db3c   ubuntu   "/bin/bash"   5 minutes ago   Exited (0) 5 seconds ago           condescending_borg
99e2e0e176e8   ubuntu   "/bin/bash"   8 minutes ago   Exited (0) 8 minutes ago           xenodochial_euler
```

This is an important lesson to learn: as soon as the program that you started in a container quits, your container stops. In contrast to *real* virtual machines, containers do not have a real operating system running, only the single application you want to run.

{{% tip %}}
**Note:** There is nothing special about the `ubuntu` image apart from it being maintained as part of the official image library.
{{% /tip %}}

## Container lifecycle

Containers in Docker can have four states:

`created`
: after a container has just been created, but has never been launched

`running`
: when it is running

`stopped`
: when it has been stopped

`paused`
: when it has been paused (rarely used)

You can easily change the states using the following commands, respectively:

- `docker start`
- `docker stop`
- `docker pause`
- `docker unpause`

Additionally, it is worth noting that `docker run` is just a shorthand for `docker create`, `docker start` and then `docker attach` to attach the console of the running container in the background.

You can, of course, start a container in the background without attaching the console using `docker run yourimage -d` 

{{% tip %}}
**Hint:** Running out of space? If you just want to clear all the unused images and containers off your machine you can use `docker system prune` to do that. But be careful, this removes a lot of stuff!
{{% /tip %}}

## Investigating the container

So let's hop into our container once more and take a look around:

```bat
C:\Users\janoszen>docker run -ti ubuntu
root@c78e8f6a792c:/#
```

### Processes

As a first command let's take a look at the processes that are running:

```bash
root@c78e8f6a792c:/# ps auwfx
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  1.0  0.1  18504  3520 pts/0    Ss   20:59   0:00 /bin/bash
root        10  0.0  0.1  34396  2896 pts/0    R+   20:59   0:00 ps auwfx
```

As you can see, in the container you only see the processes running inside the container. You don't see any of the processes running outside the container. In contrast, if you are running Docker on a Linux system, the same command on the *host system* shows you the processes inside the container:

```bash
janoszen@janoszen-laptop:~$ ps awufx
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
...
root      5197  0.1  0.1 696940 40468 ?        Ssl  21:07   0:03 /usr/bin/containerd
root     12791  0.0  0.0  10604  4620 ?        Sl   22:01   0:00  \_ containerd-shim -namespace moby -workdir /var/lib/containerd/io.con
root     12818  3.3  0.0  18508  3048 pts/0    Ss+  22:01   0:00      \_ /bin/bash
...
root      5249  0.5  0.3 912428 115696 ?       Ssl  21:07   0:16 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

As you can see the program we started inside the container is just running as a normal process, not a real virtual machine. It is *possible* to run a container as a real virtual machine, for example for security purposes if the workload is untrusted, but containers in general run all under the same operating system, which makes them very efficient.

Now, let's investigate how containers are connected. For that we will need to install the `iproute2` utility in our container:

```bash
root@c78e8f6a792c:/# apt update && apt install -y iproute2
```

### Network

Then we can take a look at the network interfaces:

```bash
root@c78e8f6a792c:/# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
...
75: eth0@if76: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

As you can see this container has it's own IP address, `172.17.0.2/16`. We can investigate the connectivity further by running the `ip route` command:

```bash
root@c78e8f6a792c:/# ip route
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```

So the container is running in a network, and it's *default gateway* is `172.17.0.1`. If you are running Docker on Linux you can simply take a look at this IP from the host:

```bash
janoszen@janoszen-laptop:~$ ip addr show dev docker0
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:7e:a1:1e:73 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7eff:fea1:1e73/64 scope link 
       valid_lft forever preferred_lft forever
```

In Docker on Windows the situation is a bit more complicated, but suffice it to say, the default network configuration is such that the containers have their own virtual network interfaces and they are connected over this virtual network
with the operating system actually running them. On the host operating system the network traffic gets send to the internet via a NAT (Network Address Translation), so that any outgoing traffic appears to come from the IP address of the host.

There are, of course, different and more complicated network setups, especially when Docker is running in Swarm mode, but we'll leave that for later.

### Disks

Finally, let's take a look at the disks:

```bash
root@c78e8f6a792c:/# mount
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/2D3RNPVHGDWACR6CZNIFGAFTWN:/var/lib/docker/overlay2/l/JHWG4KJ36FYJMIZ5ZIOO7J65F6:/var/lib/docker/overlay2/l/5QHL4NCFJSHV4GFAD35FAMNSIP:/var/lib/docker/overlay2/l/6M6LB3P7J6THDIO5DDAM5HSPEL:/var/lib/docker/overlay2/l/3ESZKSBYWFGENR6X7RHLTZRMJR,upperdir=/var/lib/docker/overlay2/193fcd23d36998f2e17108cf5317ceab0881ea3105964df6ef11e3329ee3cbed/diff,workdir=/var/lib/docker/overlay2/193fcd23d36998f2e17108cf5317ceab0881ea3105964df6ef11e3329ee3cbed/work)
...
/dev/sda1 on /etc/resolv.conf type ext4 (rw,relatime,data=ordered)
/dev/sda1 on /etc/hostname type ext4 (rw,relatime,data=ordered)
/dev/sda1 on /etc/hosts type ext4 (rw,relatime,data=ordered)
...
```

The `mount` command lists all the drives and devices that have been activated in our Linux system. If you are familiar with this command, you will find this output a bit strange. The root filesystem (`/`) is mounted not as a normal filesystem, but something called `overlay`.

This overlayfs is also an important part of Docker. It allows Docker to create a container without copying the whole image. Instead, only the files that are modified in each container from the image are stored. What's more, even images can have multiple *layers* and only the difference to the previous layer is stored. This makes for very efficient  updates of images when only one of the layers have changed.

## Creating a <del>Docker</del> <ins>container</ins> image

Ok, so we need to address the elephant in the room: it's not a Docker image. It's a container image. The reason behind this is that what once was the Docker image format has now been standardized and can be used with Docker, Kubernetes, or any other container tool.

So let's build one. In Docker we do this using a file called a `Dockerfile`. This `Dockerfile` contains a list of commands that need to be executed in order to produce the image. Having a file like this is very important because then we not only have our image that works everywhere, but we also have the *recipe* how we created our image.

Ideally, the image build process is integrated in your CI/CD pipeline, such that whenever you change your `Dockerfile`, the image is automatically rebuilt and pushed to your company registry. But, more on that later.

So, let's get started with our `Dockerfile`. Our first command will be the `FROM` command. This command will tell Docker which base image to use. For example, you could use `ubuntu`, which will pull in the latest Ubuntu, or even a specific version, such as `ubuntu:18.04`. Needless to say, you can also say `FROM scratch`, which will start from an empty container image, but this is only useful if you are working in a language like Go.

Be careful though, anyone can create and publish an image on the Docker Hub. Even official images often have flaws and may not be as well maintained as one would expect, not to mention third party images. Be conservative when using images by other people.

So, let's get started:

```Dockerfile
FROM ubuntu:18.04
```

So far so good. Let's build it, run the following in the directory you saved your `Dockerfile`:

```bat
C:\Users\janoszen\docker-test>docker build .
Sending build context to Docker daemon  2.048kB
Step 1/1 : FROM ubuntu:18.04
 ---> 775349758637
Successfully built 775349758637
```

The build command takes one compulsory parameter, the path to the `Dockerfile`, which is pointed to the current directory by specifying the dot (`.`).

This build process means that all the files in the current directory are packed up by the Docker client and are sent to `dockerd`, which is indicated by the first line (`Sending build context...`). The image is then build, which consists of one command (`FROM`).
 
However, if you are on Windows, you will get this security warning:

```bat
SECURITY WARNING: You are building a Docker image from Windows
against a non-Windows Docker host. All files and directories
added to build context will have '-rwxr-xr-x' permissions. It is
recommended to double check and reset permissions for sensitive
files and directories.
```

This warning is issued due to how Linux/Unix permissions are differend from Windows. If you build a Linux container on a Windows host, all files added via the `COPY` or `ADD` commands (more on those later) will have the executable permission. This is less than ideal since all files added will be executable. You can, of course, fix this issue by adding commands to the build process to set the permissions, but this is a good reason for using WSL instead of the Windows command line to run the build commands. Needless to say, for production uses this should be integrated in your CI/CD pipeline anyway and you shouldn't build images by hand.

Now, setting Windows issues aside how do we use this image? We can, of course, reference the image by it's ID, but that would be rather ugly:

```bat
C:\Users\janoszen\docker-test>docker run -ti 775349758637
```

Instead, we should always *tag* our images with a name:

```bat
C:\Users\janoszen\docker-test>docker build -t my-first-image .
Sending build context to Docker daemon  2.048kB
Step 1/1 : FROM ubuntu:18.04
 ---> 775349758637
Successfully built 775349758637
Successfully tagged my-first-image:latest
```

The *tag* basically names the image. It is customary to specify the image names in the format of `companyname/imagename`. This is especially important if you want to publish to the Docker Hub, as the image name will be your user or organization name, followed by the image name. If you want to push to a private registry, you will need to use the registry URL instead, such as `gcr.io/[PROJECT-ID]/[IMAGE]` for the Google Cloud Registry.

Another important aspect of tagging is *versioning*. If you don't specify a version for an image tag, it will default to `latest`. In other words the image name `ubuntu` is exactly the same as `ubuntu:latest`. However, if you are working in a serious environment you really should *version* your containers so you can easily roll back to the previous one.

So the build command should really look like this:

```bat
D:\docker-test>docker build -t mynick/my-first-image:1.0.0 .
```

Great! Our first image is done and properly tagged! If we so desire we can issue a `docker login` command to log in to the Docker Hub and `docker push` our image there. I'll leave it to you as an exercise to figure out.

Now, if you run this image you will notice that we haven't really done anything terribly useful yet. So let's change this container to run a shell script. As a first step let's create a script called `hello.sh` in the directory where your `Dockerfile` is located, with the following content:

```bash
#!/bin/bash

echo "Hello world!"
``` 

{{% warning %}}
**Be careful about Windows line breaks!** If you edit this file on Windows you may get the cryptic error `standard_init_linux.go:211: exec user process caused "no such file or directory"`. This is caused by the invisible carriage return character (`\r`) after `#!/bin/bash`. Make sure you use an editor that can save in the Linux file format with only the line feed character (`\n`) as a line ending.
{{% /warning %}}

I know, I should really come up with something more creative. Anyway, let's copy this shell script to our container during the build process:

```Dockerfile
FROM ubuntu:18.04
COPY hello.sh /hello.sh
```

This will take hello.sh into the root directory of the image. Alternatively you can use the `ADD` command too, which has the advantage over `COPY` that it can add files from a HTTP URL, and, also extract `.tar` archives in the process.

Ok, so the script has been added, but it doesn't do anything yet. As a next step we need to change the files permissions, so it can be executed:

```Dockerfile
FROM ubuntu:18.04
COPY hello.sh /hello.sh
RUN chmod +x /hello.sh
```

Finally, we need to state that instead of running an interactive shell, we should run our `hello.sh`. This is done using the `CMD` command:

```Dockerfile
FROM ubuntu:18.04
COPY hello.sh /hello.sh
RUN chmod +x /hello.sh
CMD ["/hello.sh"]
```

If we have done everything correctly, after a build our container should now run properly:

```bat
C:\Users\janoszen\docker-test>docker run my-first-image
Hello world!

C:\Users\janoszen\docker-test>
```

Note that we did not need `-ti` this time, since the program is not running interactively. Our shell script was executed, and it output what we requested.

Now, let's talk about `CMD` a little. You may see two forms of `CMD`:

- `CMD ["/your/application", "--parameter1", "--paramater2"]`
- `CMD /your/application --parameter1 --parameter2`

My recommendation is that you *always* use the former. Let's take a look what happens if you use the latter:

```bat
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.7  0.0   4628   868 ?        Ss   16:32   0:00 /bin/sh -c /your/application --parameter1 --parameter2
root         2  0.0  0.0 141108 10480 ?        S    16:32   0:00 /your/application --parameter1 --parameter2
```

If you observe carefully, your program (`hello.sh`) is running as PID 2, and the first process ID belongs to `/bin/sh`. This will result in an issue when you try to stop a container because the `SIGTERM` signal that asks a program to stop will be sent to PID 1, which will promptly ignore the signal. Your application will never receive a signal to please, finish what it does and exit gracefully. Instead Docker will then wait around 10 seconds and finally send a `SIGKILL` to the container, which will kill the application, probably in the middle of saving something.

## Manipulating images

Before we continue on, I would like to talk about the commands that you can use to manipulate your images:

`docker image ls`
: lists all images on the current host

`docker image rm imagename`
: deletes an image

`docker push` or `docker image push`
: pushes an image to a registry.

`docker pull` or `docker image pull`
: pulls an image from a registry. This is implicitly done when the image is not available locally.

`docker image save`
: saves an image as a tar file.

## Creating a webserver

Now we have the basics sorted, let's create a more complex example by <del>dockerizing</del> <ins>containerizing</ins> a webserver. We start by creating a Dockerfile based on Ubuntu:

```Dockerfile
FROM ubuntu:18.04
``` 

Next we want to install nginx. We do this by first updating the package cache, then installing nginx:

```Dockerfile
FROM ubuntu:18.04
RUN apt update
RUN apt install -y nginx
```

Now, this is not ideal because your APT package cache will be stored in the second layer, so there will be data in the container that we don't need when we run it, making the image unnecessarily large. Let's follow the [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) and pull these layers together and remove the APT cache in the same step:

```Dockerfile
FROM ubuntu:18.04
RUN apt update && \
    apt install -y nginx && \
    rm -rf /var/lib/apt/lists/*
```

Since the `RUN` command is running everything in a single step and the cache is removed immediately, the cache will not be recorded in the image.

Now we have nginx installed we can document that this server will be listening on port 80:

```Dockerfile
FROM ubuntu:18.04
RUN apt update && \
    apt install -y nginx && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80
```

Note that the `EXPOSE` command does not *publish* the service on your host machine, it simply documents that port 80 is a service that this container provides.

Now that that's sorted, let's add the `CMD`. How do we know what the proper CMD is? This is a tricky question, so let's build and start the image we have so far in interactive mode:

```bat
C:\Users\janoszen\nginx>docker build -t my-nginx:1.0.0 .
...
C:\Users\janoszen\nginx>docker run -ti my-nginx:1.0.0
```

Now let's take a look at the installed `nginx` package contents:

```bash
root@ac0b6e5fb102:/# dpkg -L nginx
/.
/usr
/usr/share
/usr/share/doc
/usr/share/doc/nginx
/usr/share/doc/nginx/copyright
/usr/share/doc/nginx/changelog.Debian.gz
```

Hm.... this seems to be only documentation. Let's actually look at what packages are installed:

```bash
root@ac0b6e5fb102:/# dpkg -l |grep nginx
ii  libnginx-mod-http-geoip        ...
ii  libnginx-mod-http-image-filter ...
ii  libnginx-mod-http-xslt-filter  ...
ii  libnginx-mod-mail              ...
ii  libnginx-mod-stream            ...
ii  nginx                          ...
ii  nginx-common                   ...
ii  nginx-core                     ...
```

Aha! So the nginx application is actually in `nginx-core` which was pulled in as a dependency. Let's look at the contents:

```bash
root@ac0b6e5fb102:/# dpkg -L nginx-core
/.
/usr
/usr/sbin
/usr/sbin/nginx
/usr/share
/usr/share/doc
/usr/share/doc/nginx-core
/usr/share/doc/nginx-core/copyright
/usr/share/man
/usr/share/man/man8
/usr/share/man/man8/nginx.8.gz
/usr/share/doc/nginx-core/changelog.Debian.gz
```

If you look at this list carefully, you will see that there is a *binary* named `/usr/sbin/nginx`. We will assume that this is the program we will want to run, so let's add the `CMD` part to the Dockerfile:


```Dockerfile
FROM ubuntu:18.04
RUN apt update && \
    apt install -y nginx && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80
CMD ["/usr/sbin/nginx"]
```

Ok, let's build and run this container, this time *publishing* port 80 on the host so we can actually access it:

```bat
C:\Users\janoszen\nginx>docker build -t my-nginx:1.0.1 .
...
C:\Users\janoszen\nginx>docker run -p 80:80 my-nginx:1.0.1
```

... aaaand the container stops immediately. Womp womp. Now you can go about spending a great deal of time debugging the issue, but let me just reveal the secret: the container stops immediately because nginx, like many server 
applications, *daemonizes*.

Daemonization means that the application goes into the background and gives you back your console control. In contrast, some applications start in the foreground, so you won't be able to use the console you started the application from 
because the application will be occupying it.

Remember what we said before? If you quit the application you start in the container, the entire container dies. This is exactly what happens with daemonized applications because daemonization involves starting a child process and exiting
the main process to give back control. So, in order to run nginx in a container we will need to stop nginx from daemonizing. After a bit of googling let's change our `CMD` to do just that:

```Dockerfile
FROM ubuntu:18.04
RUN apt update && \
    apt install -y nginx && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

Wow! That's it! If we now start this container and open [http://localhost/](http://localhost) we will see the default nginx page come up. Cool!

Now on to the meat of the matter, let's add a website. Let's create an `index.html` to that end:

```html
<html><body><h1>Hello world!</h1></body></html>
``` 

Then we can copy the file into the container:

```Dockerfile
FROM ubuntu:18.04
RUN apt update && \
    apt install -y nginx && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
COPY index.html /var/www/html/index.html
```

{{% tip %}}
**Note:** we issue the `COPY` command last so when we change the HTML file we don't need to reinstall nginx and allow Docker to use the already built layers.
{{% /tip %}}

Ok, so we have our website in the container.

{{% tip %}}
**Note:** if you `Ctrl+C` out of `docker run`, the container will stop. If you want to run the container in background, use the `-d` flag.
{{% /tip %}}

How about if we want to change the configuration file? Let's do a bit of 
digging. Let's enter the running container. To do that we will first find the container ID of the running nginx container, which we can do using `docker ps`:

```bat
C:\Users\janoszen\nginx>docker ps
CONTAINER ID   IMAGE            COMMAND                  CREATED         STATUS          PORTS                NAMES
cd19c4cf8d71   my-nginx:1.0.1   "/usr/sbin/nginx -g …"   4 minutes ago   Up 4 minutes    0.0.0.0:80->80/tcp   nostalgic_turing
```

{{% tip %}}
**Note:** You can *name* your containers using the `--name` parameter for run, but you should not rely on naming containers as you will delete and recreate them a lot.
{{% /tip %}}

Now that you have the container ID, you can actually launch a shell into the running container:

```bat
C:\Users\janoszen\nginx>docker exec -ti cd19c4cf8d71 /bin/bash
root@cd19c4cf8d71:/#
```

If your container is not running, you can alternatively override the `CMD` when running the container:

```
C:\Users\janoszen\nginx>docker run -ti my-nginx:1.0.1 /bin/bash
```

{{% tip %}}
**Hint:** don't assume a third party image contains `/bin/bash`! Sometimes the image is very (very) minimalistic to save space. See the Advanced Debugging section at the end of this article for details how to get around that!
{{% /tip %}}

This will start the container with a shell in it. Either way, we can now poke around in the container to find the configuration files. Let's go to `/etc/nginx` to take a look:

```bash
root@cd19c4cf8d71:/# cd /etc/nginx
root@cd19c4cf8d71:/etc/nginx# find
.
./fastcgi.conf
./modules-enabled
./modules-enabled/50-mod-stream.conf
./modules-enabled/50-mod-http-image-filter.conf
./modules-enabled/50-mod-http-xslt-filter.conf
./modules-enabled/50-mod-mail.conf
./modules-enabled/50-mod-http-geoip.conf
./mime.types
./sites-available
./sites-available/default
./koi-utf
./scgi_params
./win-utf
./sites-enabled
./sites-enabled/default
./conf.d
./koi-win
./modules-available
./nginx.conf
./snippets
./snippets/snakeoil.conf
./snippets/fastcgi-php.conf
./uwsgi_params
./proxy_params
./fastcgi_params
```

You can see that there are multiple configuration files. The base config file is `nginx.conf`. If you use `cat nginx.conf`, you can see that this configuration file also pulls in other config files:

```nginx
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

This is significant because other software often do this. If we keep digging around we will find that the default virtual host is located in `/etc/nginx/sites-available/default`, where we will find roughly the following content:

```bash
root@cd19c4cf8d71:/etc/nginx/sites-available# cat default 
##
# You should look at the following URL's in order to grasp a solid understanding
# ...
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;

	index index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		try_files $uri $uri/ =404;
	}
}
```

Now, if you have used `docker exec` to get into the running container you can change this config file and run
`kill -HUP 1` to reload the nginx config. Other software, like Apache, will need a different technique to reload
the config while running.

{{% tip %}}
**Note:** since we are not running in a real operating system, management services like systemd, are not running, so you can't simply restart a service.
{{% /tip %}}

Now that you have the config file changed, you can transfer the changes outside the container. Simply create a file next to your `Dockerfile` called `default` and use the `COPY` operation to move the config file in the container.

## Development environment with Volumes

Now, if you think about working on developing a website or web application, building a container image every time is a bit of a hassle, so it would be desirable to be able to dynamically edit the files.

This can be achieved with *volumes*. Volumes in the container world are basically folders that are stored outside of the container and can be used to persist data when a container is replaced. These volumes can be either a folder shared with the host machine, but in a cloud environment they can also be a block storage offered by the cloud provider.

For our development environment we will mount a folder from the host machine:

```bat
C:\Users\janoszen\nginx>docker run -p 80:80 -v C:\Users\janoszen\nginx\html:/var/www/html my-nginx:1.0.1
```

Now we can edit the files in `C:\Users\janoszen\nginx\html` and they will show up directly in the container.

In a production environment you could use volumes to store data permanently, for example you would put your MySQL data directory in a volume so you can upgrade a container (by replacing it).

## Initialization script in containers

So far we have only run a single program in a container and I strictly warned you before about using the `CMD` the wrong way. What if you wanted to run an initialization script, for example, to dynamically generate a config file?

Let's say we wanted to generate a dynamic nginx configuration as described above. First of all, we create an `init.sh` next to our `Dockerfile`. (Don't do it this way, I will explain in a second.)

```bash
#!/bin/bash

set -e

cat << EOF > /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name ${DOMAIN_NAME};
    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

/usr/sbin/nginx -g 'daemon off;'
```

Next, we modify the `Dockerfile`:

```Dockerfile
FROM ubuntu:18.04
RUN apt update && \
    apt install -y nginx && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80
COPY init.sh /init.sh
RUN chmod +x /init.sh
CMD ["/init.sh"]
COPY index.html /var/www/html/index.html
```

This will cause the init script to be run instead of nginx directly. However, this image will have the same problem as using `CMD` incorrectly: the first PID will be the shell script, not nginx, and shell scripts, by default, ignore signals sent to them. In other words, you won't be able to stop the container gracefully.

To fix this issue we will have to fix the last line of the shell script to read:

```bash
exec /usr/sbin/nginx -g 'daemon off;'
```

The `exec` stanza in this case tells Bash to, instead of running nginx as a subprocess, nginx will *replace* the process of the shell script. As a side effect, any commands written after the `exec` line will not be executed.

### Advanced scripting with `ENTRYPOINT`

Sometimes you want to make your initialization script easy to parametrize. Let's say you want to enable someone using your container like this: `docker run -ti your-nginx /bin/bash`. Unfortunately in this case your initialization script will not run. To fix that we can use `ENTRYPOINT`. This command is used in conjunction with `CMD` and the two are basically concatenated together.

Let's say we change the above script to read:

```bash
exec "$@"
```

And, then we change our `Dockerfile` to read:

```Dockerfile
ENTRYPOINT ["/init.sh"]
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

In this case the default run will launch `/init.sh /usr/sbin/nginx -g 'daemon off;'`, but if we use `docker run` with a modified `CMD` it will read `/init.sh /bin/bash`, which will still run our init script, but then hand off control to bash.

## Running multiple services in a container

The next tricky question is, how do you run multiple services in a single container? For example, how do you run nginx and PHP-FPM next to each other?

Note, that in general **it is advisable not to run more than one service in container**, but sometimes best practice takes a back seat to practicality.

Talking about practicalities, if we run nginx and PHP-FPM, which process will be PID 1? Also, what will happen if one of the two dies? Obviously these are scenarios we need to handle, and since we have these as a consideration, a simple 
shell script that runs both will not cut it.

My advice for you in this case is to use [supervisord](http://supervisord.org/), a daemon management software. It is not a 100% what we will need, but can be made to do our bidding. So let's create a Dockerfile that installs nginx, php-fpm and supervisord:

```Dockerfile
FROM ubuntu:18.04

RUN apt-get update -y && \
    apt-get install -y \
        supervisor php7.2-fpm nginx && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /run/php && \
    chown www-data:www-data /run/php && \
    mkdir /var/log/php && \
    chown www-data:www-data /var/log/php
    
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisor.conf"]
```

Now we need to create the supervisord configuration, let's call this `supervisord.conf`:

```ini
[supervisord]
nodaemon=true

[program:php7.2]
command=/usr/sbin/php-fpm7.2 -F
redirect_stderr=true
numprocs=1
autostart=true
autorestart=true
priority=10

[program:nginx]
command=/usr/sbin/nginx -g 'daemon off;'
redirect_stderr=true
numprocs=1
autostart=true
autorestart=true
priority=10

[eventlistener:quit_on_failure]
events=PROCESS_STATE_FATAL
command=/usr/local/bin/manage-supervisord
```

As you can see this is basically a configuration to run the different programs. What's important is the last bit. The `[eventlistener:quit_on_failure]` section directs supervisord to run a little program called `/usr/local/bin/manage-supervisord` every time when one of the other programs die. This is important because this little helper program will then kill supervisord, preventing nginx from running without PHP and vice versa. This program is 
can be written in Python like this:

```python
#!/usr/bin/python
import sys
import os
import signal

def write_stdout(s):
    sys.stdout.write(s)
    sys.stdout.flush()

def write_stderr(s):
    sys.stderr.write(s)
    sys.stderr.flush()

def main():
    while 1:
        write_stdout('READY\n')
        line = sys.stdin.readline()
        os.kill(1, signal.SIGTERM)
        write_stdout('RESULT 2\nOK')

if __name__ == '__main__':
    main()
    import sys
``` 

Let's save it next to the `Dockerfile` as `manage-supervisord` and modify the `Dockerfile` like this:

```Dockerfile
FROM ubuntu:18.04

# Install everything and create the directories PHP needs
RUN apt-get update -y && \
    apt-get install -y \
        locales supervisor php7.2-fpm nginx && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /run/php && \
    chown www-data:www-data /run/php && \
    mkdir /var/log/php && \
    chown www-data:www-data /var/log/php

# Set supervisord as a CMD
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisor.conf"]

# Copy the config and the manage-supervisord config
COPY supervisor.conf /etc/supervisor/supervisor.conf
COPY manage-supervisord /usr/local/bin/manage-supervisord
RUN chmod +x /usr/local/bin/manage-supervisord
```

If we now run our container we will see the processes being started successfully:

```bat
C:\Users\janoszen\nginx>docker run -p 80:80 my-nginx-php:1.0.0
2019-11-18 17:14:24,011 CRIT Supervisor running as root (no user in config file)
2019-11-18 17:14:24,013 INFO supervisord started with pid 1
2019-11-18 17:14:25,016 INFO spawned: 'quit_on_failure' with pid 8
2019-11-18 17:14:25,019 INFO spawned: 'nginx' with pid 9
2019-11-18 17:14:25,020 INFO spawned: 'php7.2' with pid 10
2019-11-18 17:14:26,050 INFO success: quit_on_failure entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2019-11-18 17:14:26,051 INFO success: nginx entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2019-11-18 17:14:26,051 INFO success: php7.2 entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```

Now, as a bit of an exercise, let's get the PHP processing actually working. First of all let's create an `index.php` file:

```php
<?php
echo("Hello world!");
```

Next, let's create the nginx config file named `default`:

```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	root /var/www/html;
	index index.php index.html index.htm index.nginx-debian.html;
	server_name _;
	location / {
		try_files $uri $uri/ =404;
	}
	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
	}
}
```

Finally, let's amend the `Dockerfile`:

```Dockerfile
FROM ubuntu:18.04

RUN apt-get update -y && \
    apt-get install -y \
        locales supervisor php7.2-fpm nginx && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /run/php && \
    chown www-data:www-data /run/php && \
    mkdir /var/log/php && \
    chown www-data:www-data /var/log/php

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisor.conf"]

COPY supervisor.conf /etc/supervisor/supervisor.conf
COPY manage-supervisord /usr/local/bin/manage-supervisord
RUN chmod +x /usr/local/bin/manage-supervisord

COPY index.php /var/www/html/index.php
COPY default /etc/nginx/sites-available/default
```

And that's it! If you did everything correctly you will see `Hello world!` on the website, being served through nginx by PHP-FPM.

## Running multiple containers with docker-compose

Since I mentioned that running PHP and nginx in a single container is not exactly ideal let's take a look at a tool that is maintained by the Docker developers, but is still a separate tool: `docker-compose`. As the name suggests, `docker-compose` is a tool to *compose* multiple containers into a single unit.

First of all, let's split up our project. Let's create two folders, `php` and `nginx` and a `Dockerfile` in each of them. The nginx `Dockerfile` will look like this:

```Dockerfile
FROM ubuntu:18.04

RUN apt-get update -y && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 80

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]

COPY default /etc/nginx/sites-available/default
```

The appropriate `default` file will look like this:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.php index.html index.htm index.nginx-debian.html;
    server_name _;
    location / {
        try_files $uri $uri/ =404;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass php:9000;
    }
}
```

Notice the `fastcgi_pass` line. It references the host name of the server running PHP-FPM. `php`, in this case, will be the host name of the PHP container, which will be accessible by the container name.

So let's create the `Dockerfile` for the PHP container. It will look very similar to the previous one, except we will need to change the PHP-FPM configuration, so one more `COPY` is added:

```Dockerfile
FROM ubuntu:18.04

# Install PHP-FPM and create the directories it needs
RUN apt-get update -y && \
    apt-get install -y php7.2-fpm && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /run/php && \
    chown www-data:www-data /run/php && \
    mkdir /var/log/php && \
    chown www-data:www-data /var/log/php

# Set PHP-FPM as CMD
CMD ["/usr/sbin/php-fpm7.2","-F"]

# Copy the pool configuration
COPY www.conf /etc/php/7.2/fpm/pool.d/www.conf
```

The pool configuration (`www.conf`) will need to be modified as we want it to listen on the network instead of a local socket so nginx can connect it:

```ini
[www]
user = www-data
group = www-data
listen = 0.0.0.0:9000
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Now that we have all our components in place, let's create the our `docker-compose.yaml` file:

```yaml
version: '3.2'
services:
    nginx:
        build: nginx
        ports:
          - "80:80"
        volumes:
          - ./src:/var/www/html
    php:
        build: php
        volumes:
          - ./src:/var/www/html
```

Before we launch, you will have to create a directory called `src` and move `index.php` into that directory as it will be mounted as a volume into both containers for development purposes.

Now you can [download docker-compose](https://docs.docker.com/compose/install/) and run `docker-compose build` and then `docker-compose up` to run your containers. This does nothing differently than your normal way of starting containers, but if a container is changed it automatically relaunches them with the newer version. You can also run `docker-compose down` to completely tear down the infrastructure.

In other words, docker-compose is just a nice way to save you from having to type a bunch of CLI commands to launch containers. Later on you will, of course, move to something like Kubernetes deployments, but this is the most basic tool to help you launch containers in conjunction and it is a very useful tool to bring even small scale production deployments up. 

## Advanced debugging

Sometimes the above techniques are not enough to debug a container. So let's go through some techniques to dissect a container.

When something doesn't work, first take a look at the output of `docker inspect`. There is a lot of detail in there, so it's easy to get lost at first, but it often contains useful information about your containers state.

Next, if you need to access some data inside the container, you can use the `docker cp` command. Let's say your application has written a log file and you have forgotten to put it on a volume. No problem, you can just grab the file
using the `docker cp` command:

```bash
docker cp 4f8e38980308:/var/log/nginx/error.log errorlog
``` 

This will copy the file to your current directory. You can, of course, also reverse the order to move a file into a container for debugging.

How about if the log files are not enough? I have previously written about [strace](/blog/debugging-applications-with-strace) as an amazing Linux debugging tool. Unfortunately if you try to run strace in a normal container you will get this:

```bash
strace: ptrace(PTRACE_TRACEME, ...): Operation not permitted
+++ exited with 1 +++
```

This has to do with the fact that Docker restricts containers from using system calls. Now, you really don't want to run a production container, such as nginx, with elevated privileges, but you can launch a companion container with elevated privileges such as this:

```bash
TARGET=main-container-id-here
docker run \
  --pid container:$TARGET \
  --net container:$TARGET \
  --cap-add NET_RAW \
  --cap-add NET_ADMIN \
  --cap-add SYS_PTRACE \
  -ti \
  ubuntu
```

This will launch the ubuntu container in the same process and network *namespace* as the main container, but with more privileges. This will let you see the processes of the other container and also run `strace` on them. Similarly, because the network namespace is also shared and you have the `NET_RAW` and `NET_ADMIN` capabilities, you can also do a `tcpdump` to capture packets.

{{% warning %}}
**Warning!** Tutorials on the Internet often advise you to run a container with `--privileged`. This basically runs your container without security constraints and is usually a pretty bad idea.
{{% /warning %}}

{{% tip %}}
**Further reading:** If you want to know how namespaces work, give my [Under the hood of Docker](/blog/under-the-hood-of-docker) post a read.
{{% /tip %}}

Let's say this all resulted in nothing and you need to take a look what's going on in the container. In this case you could also use the `docker export` command to dump the contents of a container into a tar file that you can then inspect. Alternatively, you could use `docker commit` to create a new image of a running container with all its  modifications and use it to launch a second container from it with a shell to inspect the changes. Needless to say, don't do this for a production container.

## Making containers production grade

### Security

Talking about production grade, there are a few things we haven't talked about so far, one of them being security. If you have noticed, we are always `root` in the container. Of course, Docker restricts the containers somewhat, but you have to remember that `root` is `root` and some local exploits may work despite Dockers best efforts.

So in general it is a pretty bad idea to run a container as root. So bad that some container orchestrators don't allow containers running as root by default. For a good production-grade image we should change our container to run as a
non-root user. For example, our `nginx` container could be modified:

```Dockerfile
FROM ubuntu:18.04

RUN apt-get update -y && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*
    
EXPOSE 8080

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]

COPY default /etc/nginx/sites-available/default
```

We had to change port 80 to port 8080 since ports below 1024 can only be opened if we have the `NET_ADMIN` capability, which, by default, only root has. We will also need to change our `default` file to read:

```nginx
server {
    listen 8080 default_server;
    listen [::]:8080 default_server;
    root /var/www/html;
    index index.php index.html index.htm index.nginx-debian.html;
    server_name _;
    location / {
        try_files $uri $uri/ =404;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass php:9000;
    }
}
```

The port change itself is not a problem since we can simply publish the port on the host as port 80:

```
docker run -p 80:8080 my-nginx:1.0.2
```

With this simple step we have mitigated a lot of attacks that would otherwise work.

Another important aspect of security is protecting the Docker socket. A lot of vendor images ask you to expose
`/var/run/docker.sock` in the container. While this may have legitimate uses when the image really needs to interact
with the Docker engine itself, it basically means that you are giving the container root access to your host machine.

Let's do a little exercise:

```bash
$ docker run -v \
    /var/run/docker.sock:/var/run/docker.sock:ro \
    -ti \
   ubuntu \
   /bin/bash

# We are now in a container, let's install Docker
root@fbf9a9767805:/# apt update -y && \
     apt install -y docker.io

# And here comes the escape out of the container.
root@fbf9a9767805:/# docker run --privileged \
    --net=host \
    --pid=host \
    -ti \
    -v /:/root \
    ubuntu

root@janoszen-laptop:/# cat /root/etc/shadow 
```

And I now have your password hashes on the host machine. What happened here is that I installed Docker in the container
and then used the mounted Docker socket to launch a container outside, mounting the host filesystem in a folder.
Since I have access to the Docker socket I was able to do this and I can now read the entire host filesystem. 
Furthermore, I launched a privileged container with the host networking and pid namespace, so I have complete
control over your host system. Ouch!

So be very, very, *very* careful which container you entrust with your Docker socket. Some container orchestrators, like
Kubernetes, mitigate this with RBAC (Role-Based Access Control), but in general keep in mind that having direct access
to your Docker socket, or Kubernetes Kubelet equals a really really bad day. Really bad.

### Health checks

Now, how about stability? How do we know our container is *actually* working? What if the software dies and is left
running only in a half-working state, just enough to keep the container running? Docker addresses this by implementing
health checks. These are simple scripts that periodically run and check if the service is still running.

Let's say we extend our PHP container with the following:

```Dockerfile
FROM ubuntu:18.04

RUN apt-get update -y && \
    apt-get install -y nginx curl && \
    rm -rf /var/lib/apt/lists/*

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]

COPY default /etc/nginx/sites-available/default

COPY healthcheck /usr/local/bin/healthcheck
RUN chmod +x /usr/local/bin/healthcheck
HEALTHCHECK \
    --interval=10s \
    --timeout=10s \
    --start-period=30s \
    --retries=3 \
    CMD /usr/local/bin/healthcheck
``` 

Our healthcheck script can then be something simple as:

```bash
#!/bin/bash

set -e

curl http://localhost:8080
```

This script will run inside the container every 10 seconds, testing if the webserver is still up and serving content
with a successful status code. Our `docker ps` output will show us as much:

```bash
CONTAINER ID   IMAGE            COMMAND                  CREATED          STATUS                    PORTS                NAMES
bc4535b061c0   nginxphp_nginx   "/usr/sbin/nginx -g …"   14 seconds ago   Up 12 seconds (healthy)   0.0.0.0:80->80/tcp   nginxphp_nginx_1
```

In Kubernetes you have something similar with [Liveness, Readiness and Startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

## Container orchestrators

Finally, before we wrap this up, let's talk about container orchestrators like Kubernetes and Docker Swarm. Until now we have just deployed containers on a single machine: your laptop, or some server. When we want to move to using multiple servers, suddenly a lot of problems crop up. These problems are addressed by systems like Kubernetes or Docker Swarm. Let's take a look at these problems and how they are solved.

First of all, networking. In our docker-compose example the nginx and PHP containers talked to each other using an internal network. Under the hood this was implemented by a *network bridge*. If we now want to run our containers on multiple servers, we would have to *somehow* connect their internal networks. This is quite a difficult problem to solve, especially in a cloud environment where you can't just simply bridge networks together as you see fit. Orchestrators solve this problem by employing what they dub *overlay networks*. Basically, they are glorified mesh VPNs connecting every host with every other host and simulating a network for the containers without regards to which host they are running on.

In addition to the inter-container network orchestrators also provide *ingress* networking for published services. In plain English this means that if you publish, say, port 80 for a service, it does not matter which hosts IP address you enter in your browser, your request will always end up with the correct container. This is often coupled with a cloud providers load balancer functionality, where the load balancer is configured to send traffic to all nodes participating in a Kubernetes or Swarm cluster and let the orchestrator sort out where to send the traffic. If a node for some reason goes down, the load balancers health check will discover it and not send any more requests that way.

What's more, some orchestrators like Kubernetes even allow you to define *network policies*, effectively configuring a firewall between your containers. This is especially useful when defending against attacks like [Server Side Request
Forgery](/blog/what-is-ssrf).

The other tricky aspect of containerization in the cloud is *data storage*. Data storage is a problem because if a container that stores data in a persistent fashion (such as a MySQL server) is moved from one host to another, the volume isn't automatically transferred with it. In fact, if a host server goes down the data may even be lost and the container will be restarted with an empty data directory. This can, of course, somewhat be mitigated by clustering the database server, but a better way to deal with this issue is what Kubernetes, for example, calls *persistent volumes*. These persistent volumes are a way to integrate with your cloud provider. Every time a service is launched that requires a persistent volume the cloud provider API is called and a new network block storage is provisioned. This network block storage is attached to the host the container runs on. If the container is moved, the network block storage is reattached.

Needless to say, there are many many more features that orchestrators bring with them. It is also worth mentioning that all this fancyness has its price: the system is hugely complex. Docker Swarm is one of the simpler ones, but Kubernetes is a straight up nightmare to deploy in a production-grade fashion. Yes, you can *install* it, but that won't be production grade. Therefore, it is best to leave the Kubernetes deployment to the cloud provider and use it as a service if possible.

## Integrating containers into your life

As I mentioned, it is strongly recommended to integrate your container workflow into your CI/CD pipeline. Part of the reason behind this is that containers will need to be rebuilt regularly for security updates. For a good example you could take a look at the [CI/CD configuration behind this website](https://github.com/janoszen/pasztor.at-containers/blob/master/.circleci/config.yml) for an example.

As a conclusion I would recommend that you give yourself time to learn. Sure enough, it is easy to launch your first container, but really grasping how the whole container ecosystem works and writing efficient containers takes practice.
 
