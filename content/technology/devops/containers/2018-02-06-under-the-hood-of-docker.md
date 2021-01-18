---
slug: under-the-hood-of-docker
guid: https://pasztor.at/blog/under-the-hood-of-docker/
authors:
- janos
categories:
- Technology
date: "2018-02-06T00:00:00Z"
publishDate: "2018-02-06T00:00:00Z"
summary: The runc and rkt container runtimes power Docker & co. But what powers the
  container runtimes? Read on for a deeper look into containerization technology.
images:
- posts/under-the-hood-of-docker.png
preview: posts/under-the-hood-of-docker.jpg
tags:
- Containers
- DevOps
title: Under the hood of Docker
---

It may come as a surprise to you, but containers are not a technology. They don't exist. They are actually a result
of many different technologies built into the Linux kernel. Which ones you ask? Let's take a trip down memory lane...

{{% tip %}}
**Note:** This article glosses over some historical and technological details in favor of easier understanding.
It is not meant as a guide to implementing a container runtime, as that would take MUCH more information than
a simple blog post.
{{% /tip %}}

{{% tip %}}
**Tip:** You can find working C code demos for the content below on [my GitHub](https://github.com/janoszen/demo-container-runtime/tree/master/demo).
{{% /tip %}}

## Traditional Linux resource limits

When I started out working on Linux, the only way to really limit how much resources could consume was `ulimit`, a
command that lets you set a per-process limit on things like the number of open files, etc.

Needless to say a *per process limit* is not very useful when dealing with modern day multi-tenant environments where
each user can run hundreds or even thousands of processes at the same time.

This used to be a major limitation and made general purpose hosting platforms an impossible thing to build.

## chroot: limiting filesystem access

Also a very ancient technology in Linux was a feature inherited from BSD called `chroot`, or change root. It did exactly
what the name implied, change the root filesystem. So in essence, a process could be *jailed* inside a directory and 
see that directory as its root filesystem, not knowing that there's a whole other world out there.

This is something that is widely supported in server software, even PHP-FPM can chroot itself into a directory,
preventing PHP applications from seeing outside the specified root. (Needless to say, a PHP application would need a 
number of files inside the chroot to operate properly, such as `/usr/share/zoneinfo`.)

If we were to write a little C program to demonstrate the functionality, it would look like this:

```c
chdir("/mnt/mychroot");
if (chroot("/mnt/mychroot") != 0) {
    perror("chroot /mnt/mychroot");
    return 1;
}
chdir("/");
``` 

Note the fact that the directory is changed to the directory before the chroot command, and to the root directory 
afterwards. This is done as a security precaution, because the chroot command itself does not release the current
directory pointer. In other words, if we didn't do that, it would be possible to still navigate the directory tree
outside the chroot. It is also customary to close all files (file handles) prior to the chroot to prevent accidentally
leaving a file from outside the root accessible to the program and breaking out of the chroot.

Let's expand that program a little and output the contents of the current directory:

```c
#include <stdio.h>
#include <unistd.h>
#include <dirent.h>

int main(void) {
    chdir("/mnt/mychroot");
    if (chroot("/mnt/mychroot") != 0) {
        perror("chroot /mnt/mychroot");
        return 1;
    }
    chdir("/");

    /* Read the current directory */
    DIR *dir;
    struct dirent *ent;
    if ((dir = opendir (".")) != NULL) {
        while ((ent = readdir (dir)) != NULL) {
            printf ("%s\n", ent->d_name);
        }
        closedir (dir);
    } else {
        perror ("");
        return 2;
    }
    return 0;
}
```

If you run this program on a completely empty directory, you will see that the directory is indeed empty. However,
it is important to note that the chroot syscall is only available to the user `root` and it is also not designed
to protect from a root user inside a chroot jail. It is therefore important that any program using the `chroot` system
call also switches to a non-root user after doing so.

{{% tip %}}
**Note:** Current container technology rarely uses chroot as their method of restricting filesystem access, but opt
to use things like OverlayFS, or the Linux Device Mapper directly, in conjunction with mount namespaces (see below).
{{% /tip %}}

## capabilities: making non-root users a little bit of root

Also a very important feature, originally envisioned as the POSIX IEEE 1003.1e standard, are capabilities. Many features
of the Linux kernel, such as opening a raw socket to send an ICMP echo packet (also known as ping), traditionally
required root privileges.

Needless to say, willy-nilly handing out root privileges to people or processes is a phenomenally bad idea. Luckily,
the Linux kernel offers a feature that is very similar to the aforementioned standard. Linux capabilities allow
a process running as root to switch to a non-root user, while still retaining some root features. Some of the more 
notable ones are:

- `CAP_CHOWN`: change the owner of any file.
- `CAP_KILL`: send a signal to any process (e.g. `SIGKILL`). Normally a process can only send a signal to a different process running under the same user.
- `CAP_NET_ADMIN`: manage network configuration, such as interfaces, firewalls, etc.
- `CAP_NET_RAW`: send a raw packet (e.g. ping).
- `CAP_NET_BIND_SERVICE`: open a port under 1024. (These are normally privileged ports only available to root.)
- `CAP_SETUID` and `CAP_SETGID`: change the user and group the process is running as.
- `CAP_SYS_CHROOT`: use the `chroot` system call mentioned above.
- `CAP_SYS_NICE`: change the scheduling priority of the current process.

If we want to try out these capabilities, we can use the `capsh` Linux command as root

```bash
capsh \
      --keep=1 \
      --user=janoszen \
      --caps=cap_net_raw+epi \
      -- -c "/bin/ping www.google.com"
```

If we want to accomplish this with in a C program, we can use the `capset` function.

## cgroups: process group resource limits

In 2008 Google engineers contributed an important feature to the Linux kernel: cgroups. These cgroups would let you set
resource limits not on a single process, but on a group of processes. This was an important step forward as it would
be the first time a customers programs, no matter how many processes they spawned, could be constrained to use their
allocated resources.

In fact, many modern Linux distributions expose cgroups in their filesystem under `/sys/fs/cgroup`:

```
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
```

If we want to create a cgroup, we can again use a system call, or we can use the `cgcreate` command line utility:

```bash
cgcreate -g cpu,cpuacct:/my_group
```

As you can see, we not only supply the cgroups name, but also what types of limits we want to place. Now that we have
the chroup all set up, we can move a process into it:

```bash
cgclassify -g cpu,cpuset:/my_group 1234
```

Similarly, deletion can be done using the `cgdelete` command. If we were to limit the CPU cores this process can run on
to core 2, 3, and 4, we would do so like this:

```bash
echo "2,3,4" > /sys/fs/cgroup/cpuset/my_group/cpuset.cpus
```

A few notable examples of limits that can be set include:

- `cpuset.cpus`: sets the CPU cores the process can run on.
- `cpu.shares`: the amount of time slots this process gets on the CPU. CPU time is divided according to the shares.
- `cpu.rt_runtime_us`: the exact amount of microseconds of CPU time this process gets per time period.
- `cpu.rt_period_us`: the time period for the aforementioned limit.
- `memory.limit_in_bytes`: total memory available to the cgroup, including file cache.
- `memory.memsw.limit_in_bytes`: physical and virtual memory limit, excluding caches.
- `blkio.throttle.read_bps_device`: bytes per second limit on disk reads.
- `blkio.throttle.read_iops_device`: IO per second limit on disk reads.
- `blkio.throttle.write_bps_device`: bytes per second limit on disk writes.
- `blkio.throttle.write_iops_device`: IO per second limit on disk writes.

## Namespaces

This is where it begins to be really interesting. Namespaces have been around, but haven't really been useful until
recently. But what are they?

Let's take an example. Normally, the Linux kernel would launch one network stack for one machine, so every process
has access to the same IP addresses, etc. However, using namespaces, it is able to launch a different network stack
for a certain process on, say, a virtual network card. So for all intents and purposes, that process does not see the
"standard" IP address of the computer running it, but rather has its own, dedicated network card.

This can, of course, not only be done for network, but rather for:

- cgroups
- user IDs (UID)
- network
- mount points
- interprocess communication (IPC)
- process IDs (PID)
- hostname (UTS)

Namespaces can be started and managed using the `clone()`, `setns()` és `unshare()` system calls. If we look at an easy
to demonstrate example, a process inside a PID namespace would not see any processes running outside of it:

```bash
$ newpid bash
$ ps auwfx
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4188    84 pts/0    S    09:20   0:00 newpid bash
root         2  2.6  0.6  22372  3436 pts/0    S    09:20   0:00 bash
root        14  0.0  0.2  18440  1196 pts/0    R+   09:20   0:00  \_ ps auwfx
```

If you inspect the namespaces above, we *almost* have all makings of the container system today.

## seccomp: protecting system calls

You may have noticed that so far we haven't protected one thing: which process is allowed to make which system call.
Some system calls, of course, are limited to root, but a process running as root inside a "container" is still root, so
it can still reboot the machine. Setting the fact aside that you really really (really) shouldn't run stuff as root,
even in containers, this can still be a problem.

Seccomp solves just that problem: it allows you to deny a process a list of system calls, which is a very useful tool if
you want to prevent a "containerized" process of doing something it is not supposed to.

Let's look at an example. If you like pain, you can use the `prctl()` directly, but for everyone else I would
suggest using the `seccomp.h` include in C. Applying a seccomp profile starts with the `seccomp_init` call, which takes
a parameter that specifies the default operation mode:

- `SCMP_ACT_KILL`: if a process violates the seccomp rule, it is killed immediately.
- `SCMP_ACT_TRAP`: if a process violates the seccomp rule, it is sent a `SIGSYS` signal, and may then handle the situation gracefully.
- `SCMP_ACT_ERRNO`: if a process violates the seccomp rule, it is sent the specified error number (useful for simulating normal behavior).
- `SCMP_ACT_TRACE`: if a process violates the seccomp rule and is being traced, the tracing process will be notified.
- `SCMP_ACT_LOG`: if a process violates the seccomp rule, the syscall will be logged. This is useful for debugging a seccomp profile.
- `SCMP_ACT_ALLOW`: allows a specific system call (or all if used in `seccomp_init`)

Following the `seccomp_init`, we can now call a series of `seccomp_rule_add` calls to specify overrides to the default.
Once all rules are added, the `seccomp_load` command can be used to apply the seccomp profile. The process is now jailed
in terms of syscalls.

Taking a look at a full example, the following code can be compiled using `gcc test.c -lseccomp` and will result in a
`SIGSYS` when it reaches the `printf` line:

```c
#include <stdio.h>
#include <stdlib.h>
#include <seccomp.h>
#include <fcntl.h>
#include <errno.h>

int main()
{
    int ret;
    scmp_filter_ctx ctx;

    ctx = seccomp_init(SCMP_ACT_KILL);
    if (ctx == NULL) {
        printf("Failed initializing seccomp\n");
        exit(1);
    }
    ret = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
    if (ret < 0) {
        printf("Failed allowing exit_group via seccomp\n");
        exit(1);
    }
    ret = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(mprotect), 0);
    if (ret < 0) {
        printf("Failed allowing mprotect via seccomp\n");
        exit(1);
    }
    ret = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(munmap), 0);
    if (ret < 0) {
        printf("Failed allowing munmap via seccomp\n");
        exit(1);
    }
    ret = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(brk), 0);
    if (ret < 0) {
        printf("Failed allowing brk via seccomp\n");
        exit(1);
    }
    ret = seccomp_load(ctx);
    if (ret < 0) {
        printf("Failed loading seccomp\n");
        exit(1);
    }

    printf("Hello world!\n");
    return 0;
}
```

## AppArmor and SELinux: Mandatory Access Control Systems

Mandatory Access Control systems are different than the ones discussed before: they won't work on a process level, but
rather on a system level, and they prevent a certain *binary* from doing things it shouldn't. For example, you could
set the program `/bin/ping` to be allowed to use the `NET_RAW` capability, regardless of which process started it. This
could be achieved in an AppArmor profile that looks like this:

```
/bin/ping {
    ...
    capability net_raw,
    ...
}
```

A detailed discussion of AppArmor or SELinux would take an article of their own, but suffice it to say, these are very
powerful tools to further restrict programs in what they are capable of doing.

## Putting it all together

Docker and other containerization technologies utilize what's called a *container runtime*. These runtimes are the
platform-dependent parts of the containerization technology as opposed to the platform independent parts that deal with
handling the images, etc.

The two main runtimes used today are `runc` by Docker and `rkt` by the CoreOS folks. These utilities take a root
filesystem image and a configuration for all of the above features (and a bit more) and run a container with it.

If we take a look at the runc configuration JSON file, we see things like this:

```json
...
"capabilities": [
        "CAP_AUDIT_WRITE",
        "CAP_KILL",
        "CAP_NET_BIND_SERVICE"
],
...
```

Again, we could go into much more detail in how runc or rkt work, but you get the picture: the container runtime is a
tool that deals with all of the above restrictions, whereas Docker and the other container engines are responsible for
*creating* this file and the associated root filesystem image in the first place. Luckily, the 
[Open Container Initiative](https://www.opencontainers.org/) is working hard to ensure interoperability between the
container technologies and runtimes, so a fracture ecosystem, it seems, has been narrowly avoided.

I hope it is clear that nobody should attempt to build their own container runtime, or implement the above restrictions
into their application without some serious research into the topic. It took Docker years to get to its current state,
so implementing anything like this in a commercial project is a futile attempt. Nevertheless, when confronted with
advanced configuration of LXC, Docker or other container technologies, it is good to know what's behind the curtain, so
I hope you enjoyed this little peek into what Docker does under the hood.
  