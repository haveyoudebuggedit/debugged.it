---
slug: debugging-applications-with-strace
guid: https://pasztor.at/blog/debugging-applications-with-strace/
authors:
- janos
categories:
- Technology
date: "2012-04-19T00:00:00Z"
publishDate: "2012-04-19T00:00:00Z"
summary: There are times, when we get an application and need to find out what it
  does fast. We don’t have the time to read the source code. Fortunately there are
  multiple tools to our rescue, one of which is the strace Linux utility. strace means
  system call trace, it shows us every system call the application does, such as opening
  or reading a file, writing data to a network socket. It’s not a magic pill, it won’t
  show the internal working of the application, but it’s still very useful to find
  out what it does externally (IO operations and such).
tags:
- DevOps
title: Debugging applications with strace
---

The basic use of strace is to run an application directly:

```bash
strace ls /proc
```

You’ll get is a list of system calls the process has done:

```
execve("/bin/ls", ["ls", "/proc"], [/* 19 vars */]) = 0
brk(0)                                  = 0x1097000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0bb0334000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY)      = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=22620, ...}) = 0
mmap(NULL, 22620, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f0bb032e000
close(3)                                = 0
open("/lib64/librt.so.1", O_RDONLY)     = 3
...
```

What you will most probably interested in are the stat, open, read, write, readv, writev, recv, recvfrom, send and sendto operations. Adding a filter for those makes the strace much more useful:

```bash
proxy / # strace -e trace=open,read,write,readv,writev,recv,recvfrom,send,sendto ls /proc
open("/etc/ld.so.cache", O_RDONLY)      = 3
open("/lib64/librt.so.1", O_RDONLY)     = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340\"\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libcap.so.2", O_RDONLY)    = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\26\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libacl.so.1", O_RDONLY)    = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340#\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libc.so.6", O_RDONLY)      = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000\356\1\0\0\0\0\0"..., 832) = 832
open("/lib64/libpthread.so.0", O_RDONLY) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\300\\\0\0\0\0\0\0"..., 832) = 832
open("/lib64/libattr.so.1", O_RDONLY)   = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0 \26\0\0\0\0\0\0"..., 832) = 832
open("/proc", O_RDONLY|O_NONBLOCK|O_DIRECTORY|O_CLOEXEC) = 3
write(1, "1      13239  14532  22092  2209"..., 1761      13239  14532  22092  22094  22096  247  447  574  8517	  cpuinfo  fairsched   filesystems  kmsg     locks    mounts  self  swaps  sysrq-trigger  uptime	     version  vz
) = 176
write(1, "13238  14496  14533  22093  2209"..., 16913238  14496  14533  22093  22095  22097  446  559  649  cmdline  devices  fairsched2  fs	    loadavg  meminfo  net     stat  sys    sysvipc	  user_beancounters  vmstat
) = 169
```

As you can see, the strings are truncated to a given length, so you don’t see everything. Adding the `-s` parameter lets you specify the string truncation length:

```bash
strace -s 999 ls /proc
```

As a result you’ll now see a lot more information at the cost of readability. This about covers directly running the application.

There however is one **major pitfall** with directly runnin strace: SUID doesn’t work. If you want to run a SUID binary like su, sudo, suexec, etc it simply won’t work as expected.

So what if we want to strace an already existing process? Nothing easier than that, the `-p` flag does just that:

```bash
strace -p 14681
```

strace will attach the process and show you what it does. Keep in mind that printing out a lot of information slows the process down.

If you need to track multiple processes (for example multiple processes of a webserver) you can just add multiple `-p` flags:

```bash
strace -p 20557 -p 20558 -p 2055
```

An other way to follow multiple processes is using the `-f` flag to follow forks. (Forks are when a process spawns an other process.)

```bash
strace -f -p 20557
```

This about covers the basic usages of strace. To get more detailed information about a specific system call, use `man syscallname`. Needless to say you can only attach processes that run under your username if you are not root.

## Examples

Let’s see an example. I have created a _hello world_ application and want to trace it on my local Apache installation. First I get the process ID’s of Apache:

```bash
root@janoszen-imac:~# ps aux | grep apache | grep -v grep 
root      6883  0.0  0.2 132136  8560 ?        Ss   13:18   0:00 /usr/sbin/apache2 -k start
www-data  6888  0.0  0.1 132616  6732 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6889  0.0  0.1 132200  5696 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6890  0.0  0.1 132600  6712 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6891  0.0  0.1 132160  5056 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6892  0.0  0.1 132160  5056 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6894  0.0  0.1 132160  5056 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6895  0.0  0.1 132160  5056 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
www-data  6896  0.0  0.1 132160  5056 ?        S    13:18   0:00 /usr/sbin/apache2 -k start
```

The second columns contains the process ID’s (PID’s), which have to be used for strace. To create a list for strace you can use the following line:

```bash
root@janoszen-imac:~# ps aux | grep apache | grep -v grep | awk ' { print $2 } ' | xargs -i echo -n ' -p {}'
 -p 6883 -p 6888 -p 6889 -p 6890 -p 6891 -p 6892 -p 6894 -p 6895 -p 6896
```

Next step, run the strace and do a single request:

```bash
root@janoszen-imac:~# strace -e trace=open,read,write,readv,writev,recv,recvfrom,send,sendto -s 999 -p 6883 -p 6888 -p 6889 -p 6890 -p 6891 -p 6892 -p 6894 -p 6895 -p 6896
Process 6883 attached - interrupt to quit
Process 6888 attached - interrupt to quit
Process 6889 attached - interrupt to quit
Process 6890 attached - interrupt to quit
Process 6891 attached - interrupt to quit
Process 6892 attached - interrupt to quit
Process 6894 attached - interrupt to quit
Process 6895 attached - interrupt to quit
Process 6896 attached - interrupt to quit
[pid  6892] read(8, "GET /test.php HTTP/1.1\r\nHost: stuff.localhost\r\nUser-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:11.0) Gecko/20100101 Firefox/11.0\r\nAccept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8\r\nAccept-Language: en-us,en;q=0.5\r\nAccept-Encoding: gzip, deflate\r\nConnection: keep-alive\r\nReferer: http://stuff.localhost/\r\nCache-Control: max-age=0\r\n\r\n", 8000) = 361
[pid  6892] open("/.htaccess", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
[pid  6892] open("/var/.htaccess", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
[pid  6892] open("/var/www/.htaccess", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
[pid  6892] open("/var/www/stuff.localhost/.htaccess", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
[pid  6892] open("/var/www/stuff.localhost/htdocs/.htaccess", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
[pid  6892] open("/var/www/stuff.localhost/htdocs/test.php/.htaccess", O_RDONLY|O_CLOEXEC) = -1 ENOTDIR (Not a directory)
[pid  6892] open("/var/www/stuff.localhost/htdocs/test.php", O_RDONLY) = 9
[pid  6892] open("/dev/urandom", O_RDONLY) = 9
[pid  6892] read(9, "\255\263\316-\306C\273\2", 8) = 8
[pid  6892] open("/dev/urandom", O_RDONLY) = 9
[pid  6892] read(9, "9\326|\367v2\21\315", 8) = 8
[pid  6892] open("/dev/urandom", O_RDONLY) = 9
[pid  6892] read(9, "5\5\305,`J\201 ", 8) = 8
[pid  6892] writev(8, [{"HTTP/1.1 200 OK\r\nDate: Thu, 19 Apr 2012 11:29:29 GMT\r\nServer: Apache/2.2.20 (Ubuntu)\r\nX-Powered-By: PHP/5.3.6-13ubuntu3.6\r\nVary: Accept-Encoding\r\nContent-Encoding: gzip\r\nContent-Length: 32\r\nKeep-Alive: timeout=5, max=100\r\nConnection: Keep-Alive\r\nContent-Type: text/html\r\n\r\n", 273}, {"\37\213\10\0\0\0\0\0\0\3", 10}, {"\363H\315\311\311W\10\317/\312IQ\4\0", 14}, {"\243\34)\34\f\0\0\0", 8}], 4) = 305
[pid  6892] read(8, 0x7fa392c8e048, 8000) = -1 EAGAIN (Resource temporarily unavailable)
[pid  6892] write(6, "stuff.localhost:80 127.0.0.1 - - [19/Apr/2012:13:29:29 +0200] \"GET /test.php HTTP/1.1\" 200 305 \"http://stuff.localhost/\" \"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:11.0) Gecko/20100101 Firefox/11.0\"\n", 200) = 200
```

As you can see, the first read() will show the request from the browser. Next the server reads the `.htaccess` files, which points out why using .htaccess files is a bad idea in a high performance solution. Finally the server reads the PHP file (and passes control to PHP). PHP then does some initialization on its own, that’s why it accesses `/dev/urandom`. Next you see the server write the response to the client and write to the access log.

As you can see, strace gives you a good idea about what the application does externally. I use strace with a wide variation of software, most of the time debugging PHP with it when the interpreter isn’t very helpful. I hope it helps getting on the trail of errors quickly. (Let me know, if it does.)
