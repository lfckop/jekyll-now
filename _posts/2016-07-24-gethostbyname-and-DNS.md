---
layout: post
title: gethostbyname与DNS
---

# 一句话总结

DNS在Linux上的执行主要由glibc库函数`gethostbyname`与`gethostbyaddr`来完成，通过`strace`跟踪可知`gethostbyname`的执行流程如下：

1. 连`/var/run/nscd/socket`，通过`nscd`进程缓存的内容来获取。`nscd`会实时监测`/etc/hosts`和`/etc/resolv.conf`文件内容的变化并缓存，具体请参考`/etc/nscd.conf`中的说明。
 
2. 如果没有`nscd`服务，则通过`/etc/nsswitch.conf`中的`"hosts: files dns"`来决定获取顺序，先读`/etc/hosts`，否则就读`/etc/resolv.conf`走`DNS`。

注：关于DNS与`dig`命令的介绍请阅读[阮一峰](http://www.ruanyifeng.com/blog/)老师的文章[DNS原理入门](http://www.ruanyifeng.com/blog/2016/06/dns.html)。

# strace追踪gethostbyname

#


