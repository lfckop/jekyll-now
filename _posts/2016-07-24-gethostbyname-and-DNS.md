---
layout: post
title: gethostbyname与DNS
---

# 一句话总结

域名查询(Domain Name Query)在Linux上的执行主要由glibc库函数`gethostbyname`与`gethostbyaddr`来完成，通过`strace`跟踪可知`gethostbyname`的执行流程如下：

1. 连接socket文件`/var/run/nscd/socket`，通过`nscd`进程缓存的内容来获取。`nscd`会实时监测`/etc/hosts`和`/etc/resolv.conf`文件内容的变化并缓存，具体请参考`/etc/nscd.conf`中的说明。
 
2. 如果没有`nscd`服务，则通过`/etc/nsswitch.conf`中的配置项`"hosts: files dns"`来决定获取顺序，先读`/etc/hosts`，否则就读`/etc/resolv.conf`向`DNS`服务器发出请求。

注：关于DNS与`dig`命令的介绍请阅读[阮一峰](http://www.ruanyifeng.com/blog/)老师的文章[DNS原理入门](http://www.ruanyifeng.com/blog/2016/06/dns.html)。

# `strace`跟踪`gethostbyname`

先介绍下我的系统环境：

```bash
# cat /etc/centos-release 
CentOS release 6.6 (Final)

# uname -a
Linux vm-10-11-174-154 2.6.32-926.504.30.3.letv.el6.x86_64 #1 SMP Mon Aug 3 16:29:31 CST 2015 x86_64 x86_64 x86_64 GNU/Linux

# ll /lib64/libc.so.6
lrwxrwxrwx 1 root root 12 3月  25 2016 /lib64/libc.so.6 -> libc-2.12.so*

# /lib64/libc.so.6 
GNU C Library stable release version 2.12, by Roland McGrath et al.
Copyright (C) 2010 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 4.4.7 20120313 (Red Hat 4.4.7-16).
Compiled on a Linux 2.6.32 system on 2016-02-16.
Available extensions:
	The C stubs add-on version 2.1.2.
	crypt add-on version 2.1 by Michael Glad and others
	GNU Libidn by Simon Josefsson
	Native POSIX Threads Library by Ulrich Drepper et al
	BIND-8.2.3-T5B
	RT using linux kernel aio
libc ABIs: UNIQUE IFUNC
For bug reporting instructions, please see:
<http://www.gnu.org/software/libc/bugs.html>.
```

在**没有**配置`nscd`的服务器上，用`strace`追踪`gethostbyname`，其大致流程摘录如下(省略的部分用`......`表示)。库函数`gethostbyname`的使用可参考[该页面](https://support.sas.com/documentation/onlinedoc/sasc/doc750/html/lr2/ztbyname.htm)。

``` c
# strace ./gethostbyname "lecloud.com"
execve("./gethostbyname", ["./gethostbyname", "lecloud.com"], [/* 32 vars */]) = 0
brk(0)                                  = 0x741000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fa832c1e000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY)      = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=61583, ...}) = 0
mmap(NULL, 61583, PROT_READ, MAP_PRIVATE, 4, 0) = 0x7fa832c0e000
close(4)                                = 0
open("/lib64/libc.so.6", O_RDONLY)      = 4
......
open("/etc/resolv.conf", O_RDONLY)      = 4
......
socket(PF_FILE, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 4
connect(4, {sa_family=AF_FILE, path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(4)                                = 0
socket(PF_FILE, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 4
connect(4, {sa_family=AF_FILE, path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(4)                                = 0
open("/etc/nsswitch.conf", O_RDONLY)    = 4
......
open("/etc/host.conf", O_RDONLY)        = 4
......
open("/etc/hosts", O_RDONLY|O_CLOEXEC)  = 4
......
socket(PF_INET, SOCK_DGRAM|SOCK_NONBLOCK, IPPROTO_IP) = 4
connect(4, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
gettimeofday({1481080561, 158097}, NULL) = 0
poll([{fd=4, events=POLLOUT}], 1, 0)    = 1 ([{fd=4, revents=POLLOUT}])
sendto(4, "Zq\1\0\0\1\0\0\0\0\0\0\7lecloud\3com\0\0\1\0\1", 29, MSG_NOSIGNAL, NULL, 0) = 29
poll([{fd=4, events=POLLIN}], 1, 5000)  = 1 ([{fd=4, revents=POLLIN}])
ioctl(4, FIONREAD, [77])                = 0
recvfrom(4, "Zq\201\200\0\1\0\3\0\0\0\0\7lecloud\3com\0\0\1\0\1\300\f\0"..., 1024, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.1")}, [16]) = 77
close(4)                                = 0
exit_group(0)                           = ?
```




要想查看某个程序是否调用`gethostbyname`，只需用`ltrace`命令跟踪一下查看其库调用即可：

```bash
# ltrace ping -c 1 baidu.com 2>&1 | grep gethostbyname
gethostbyname("baidu.com")                       = 0x7f21dfaf16e0
```

介绍nscd，dns cache server

注：在`man gethostbyname`中有一句：

> The **gethostbyname()** and **gethostbyaddr()** functions are obsolete.  Applications should use **getaddrinfo(3)** and **getnameinfo(3)** instead.

# 其它

## `nscd`

## `dnsmasq`

## `dig`


