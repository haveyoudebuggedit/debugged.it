---
slug: docker-101-anatomy
guid: https://pasztor.at/blog/docker-101-anatomy/
authors:
- janos
categories:
- Technology
date: "2018-02-08T00:00:00Z"
publishDate: "2018-02-08T00:00:00Z"
summary: Docker is (mostly) used to run Linux, but in order to successfully create a Docker image, we must first understand how Linux works.
images:
- posts/docker-101-anatomy.png
preview: posts/docker-101-anatomy.jpg
tags:
- Containers
- DevOps
title: 'Docker 101: Linux Anatomy'
---

If you are used to Windows system administration, you may be used to only being able to clone a system using
image-level backups. Linux is much different, however. You can simply copy all files on a system to a different machine,
adjust a few settings and boot the machine as if nothing happened.

This is an important concept to grasp, since container technology relies heavily on being able to pack up a Linux
installation and shipping it. In fact, the standard OCI (Open Container Initiative) image used nowadays is basically a
packed-up Linux with an additional config file in JSON format.

{{% tip %}}
**Tip**: if you wish to learn more about how containers work internally, please read the previous article,
[Under the hood of Docker](/blog/under-the-hood-of-docker). 
{{% /tip %}}

{{% tip %}}
**Note**: this article glosses over some of the intricacies of how a CPU and the kernel work for easier understanding.
If you wish to learn more about operating systems, I recommend the book Modern Operating Systems by Andrew S. Tanenbaum for further study.
{{% /tip %}}

## The CPU

In high school you have probably learned that the core component of any modern computer is the Central Processing Unit 
(CPU). What they don't teach you all that often is how a CPU actually works.

From a programmers perspective, the CPU has a set of very low-level instructions, such as `MOV` for moving data from
one memory address to the other, `JMP` to jump to a certain memory address and execute the code there, etc. This is
called an assembly language and it translates 1:1 to machine readable binary code. High level languages, like C, C++,
etc abstract this of course, and even higher level languages like Java have their own virtual machine (JVM) that
has have their own, high level assembly language. (So yes, you can build a CPU that executes Java byte code directly.)

However, if you look carefully, CPU assembly instructions can be very dangerous. A program could simply specify
the memory address of another program and cause all sorts of havoc. A programmer writing code that is able to access all
memory has to be extra careful not to mess up. Back in the MS-DOS days this was the only way programs could operate.

This was not a big problem in MS-DOS since it only ran one program in parallel (save for a few drivers,
[TSRs](https://en.wikipedia.org/wiki/Terminate_and_stay_resident_program)), but more modern systems needed some way to
protect programs from each other. That's why the then-modern 80286 CPU introduced and the
[80386 expanded upon](https://en.wikipedia.org/wiki/Virtual_8086_mode) a feature called **virtual memory**. The CPU
could now operate in multiple modes. A part of the operating system would run in the so-called ring 0 and had access to
all memory space. This was also known as kernel mode. User programs on the other hand would run in the
so-called user mode or ring 3.

Running in user mode meant that each program would have its own virtual memory space, which the CPU would then translate
to a physical memory address, or if that chunk of memory was moved to a swap file, it would interrupt the program
execution with a page fault and call a specific subroutine in the operating system to load that piece of memory.

Similarly, if a program wanted to access, say, a file, it needed to talk to the operating system kernel to do that since
it no longer had direct access to the underlying hard drive. This was achieved using the `TRAP` instruction. The `TRAP`
instruction, similar to a page fault, would interrupt the program execution and call an operating system subroutine to
achieve the functionality it desired.

### Virtualization

The CPU has one more feature that is a more recent addition: hardware virtualization. Instead of the kernel intercepting
requests to memory, modern CPUs can also assist virtualization. Requests to memory, IO, and other devices can be
intercepted by the so-called hypervisor, allowing you to run multiple kernels in parallel.

## The Linux boot process

When a machine boots, the first program being executed on the CPU is the BIOS program, or on more modern systems the
UEFI program. This program is responsible for doing a low-level hardware check (such as detecting stuck keys) and
then it looks at the hard drive to find the **bootloader**.

The bootloader is a little piece of code that is responsible for loading the operating system kernel itself, and it
has its origins in how the BIOS worked.

On BIOS systems the boot loader was a very primitive program written into the so-called Master Boot Record of the 
drive. This consisted of a few magic bytes (telling the BIOS that there is a valid boot loader installed) and then 512
bytes of executable code. Since the MBR was only so large, the whole kernel could not fit in there and that's 
one of the reasons why the boot loader had to be separated out.

It is also worth mentioning that the BIOS had no concept of filesystems. It could really only read and execute
a very specific address on the disk. Older boot loaders like LILO fit into this 512 byte constraint and the only thing
they could do was, again, execute a tiny piece of code from a specified memory address. Similar to the BIOS, LILO did
not have a concept of filesystems, so every time the Linux kernel was updated, LILO had to be updated as well.

More modern boot loaders like Grub utilized a trick, however. On traditional (MS-DOS) partitioning tables there is a 2
MB gap between the Master Boot Record and the first partition. (This is the place where boot viruses liked to write
their code.) Grub utilizes this space to give it more space for code, in contrast to LILO, Grub can read a wide range
of filesystems.

This means that in Grub one only has to specify which *file* the Linux kernel is in and it is no longer dependent
on the actual location on the disk.

With (U)EFI much of this magic is now obsolete, since UEFI can read VFAT filesystems. On UEFI systems, Grub is no longer
installed into the Master Boot Record, but on a special UEFI boot partition formated to VFAT instead of the traditional 
Linux EXT2/3/4.

However, both on BIOS and UEFI, the result was the same. The Linux kernel would be executed on the CPU and had exclusive
control over it, save for built-in features like the Intel Management Engine. 

## The init program

When booting, the Linux kernel has a bunch of stuff to do, like initializing hardware, etc. These steps can be tracked
using the `dmesg` command. However, once that is finished, it is time to switch the CPU user mode and execute the first
user space program.

This program in Linux is called the *init* program and gets the process ID 1. Without this program the Linux kernel is
pretty much useless because it won't do anything. The init program is responsible for starting all kinds of services,
like your system log, webserver, or even your graphical interface.

If you wanted, you could write this init program yourself. It could even be something as simple as a shell script.
However, this responsibility is not to be taken lightly as the init program has a few responsibilities:

- If a process terminates, its child processes will be re-parented to the init program.
- When a process terminates, it must properly clean it up, so it doesn't stay in *zombie* mode. This is true not only
  for programs the init program itself started, but also for the re-parented child processes.
- It must handle signals correctly.

A few notable examples of init systems include OpenRC and Systemd, which are both widely used in various Linux 
distributions.

## Containerization

Containers are different from virtualization in that they don't have a separate kernel. Every container runs
on the same Linux kernel, and uses features in the kernel itself to achieve separation. This has been explained in
great detail in my [previous article](/blog/under-the-hood-of-docker).

When a container starts, the Linux kernel *simulates* a new environment, where it gives the newly started process
a virtual process ID 1, its own network stack and even its own virtual or physical disk. This means that running 
containers can be much more efficient than a fully virtualized environment, since the container only requires minimal
amounts of system resources for the parts of the Linux kernel that are run for the purposes of separation. You could,
in fact, run thousands of containers on a single commodity machine.

Naturally, you could use this level of separation to run a full Linux user space system, including your average set of
tools, such as a syslog server, a webserver, etc. This is what LXC does, for example.

Note that some software does require exclusive hardware control and cannot be separated out, so you will have trouble
running:

- systemd (the init system)
- a full graphical environment on your video card
- etc.

However, running a full Linux environment for every container is a bit wasteful, since most programs started in a
container tend to be rather self-contained and don't require a great deal of tooling. So instead the world has moved
in the direction of Docker, where you would build a minimal Linux environment and **start only the program you intend to
run as your init program**.

However, if you want to run more than one program inside a container, you need to be careful since your init program
then needs to fulfill the obligations outlined above.

{{% warning %}}
**Danger:** shell scripts are tricky to get right as init programs. Instead, you should use something like supervisord if you need to run multiple applications in your container. The rkt container runtime has a workaround for this, but it's not without its drawbacks.
{{% /warning %}}

In all cases, your program must handle signals correctly. (So it must stop if it receives a `SIGTERM`, for example.)

{{% warning %}}
**Danger:** if your programs does not handle signals correctly, it will be killed without the ability to shut down properly when the container is stopped.
{{% /warning %}}

### Docker on other systems

So far we have only talked about containers running on Linux systems. Docker, however, supports other systems like 
Windows or MacOS as well. On these systems you don't get the benefit of having a shared Linux kernel, instead every
container is run on a full virtualization environment (HyperV, VirtualBox). This also means that some features are not
available.

## Building a container

Now that we know how the Linux system works, how do we build a container? First of all, we need a working Linux
userspace. This will include some system libraries and possibly some tools. A very minimalistic environment could be
achieved by using a chroot building toolkit that analyzes the library dependencies and puts everything into a folder.

We can then pack this folder into a .tar.gz achive, write an OCI-compliant container configuration file for it and 
simply run it using `runc` or `rkt`. In other words, you could simply take an average Linux machine, copy everything
into a compressed archive, add a config file and run it in a container.

Note that this way of creating containers is not self-documenting, and not really replicable, so you shouldn't do that.
Instead, Docker gives us a much more advanced set of tools, where we can describe in a script how our container should
be built. This not only gives us the option to test our build process, but also documents it at the same time.

However, that is the topic of our next article. Until then, stay tuned.

{{% tip %}}
**Recommended exercise**: to better understand how Linux works, I would recommend installing a Gentoo Linux from the minimal CD/DVD. Since everything has to be done and compiled manually, it gives you a great insight into how Linux works under the hood. 
{{% /tip %}}