---
layout: post
title: 关于Linux文件目录/proc
---

# 一句话总结
通过`strace`命令跟踪可知，经常使用的`ps`、`top`、`free`等命令都是通过读`/proc`目录下的文件获得数据然后加以处理和展示。`/proc`目录是`procfs`文件系统的挂载点，`procfs`作为内核中数据结构的访问接口，提供系统的运行时信息。

# strace跟踪
`free -m`命令会读文件`/proc/meminfo`，示例如下：

``` bash
# strace free -m 2>&1 | grep "/proc/meminfo"
open("/proc/meminfo", O_RDONLY)         = 4
```

`lscpu`命令会读文件`/proc/cpuinfo`和`/sys/devices/system/cpu/`目录下的文件。

`ps`和`top`命令会读`/proc/sys/kernel/pid_max`和`/proc/$pid/{stat, status, cmdline}`等文件。

还有很多其它的命令在此不再赘述，感兴趣的用`strace`跟踪一下命令的执行即可。



# /proc介绍
在此引用相关文档的解释和说明，感兴趣的同学可根据对应文档深入了解。

<https://github.com/torvalds/linux/blob/master/Documentation/filesystems/proc.txt>

> The proc  file  system acts as an interface to internal data structures in the
kernel. It  can  be  used to obtain information about the system and to change
certain kernel parameters at runtime (sysctl).

&nbsp;

<http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html>

> You might wonder how you can see details of a process that has a file size of 0. It makes more sense if you think of it as a window into the kernel. The file doesn't actually contain any data; it just acts as a pointer to where the actual process information resides.

&nbsp;

`man proc`

> The proc file system is a pseudo-file system which is used as an interface to kernel data structures. It is commonly mounted at /proc. Most of it is read-only, but some files allow kernel variables to be changed.

