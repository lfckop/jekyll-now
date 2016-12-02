---
layout: post
title: NoClassDefFoundError与类加载
---

# 一句话总结
若一个类在类加载过程中的任一阶段出错，随后对该类的任何主动引用都只会抛出`java.lang.NoClassDefFoundError`异常，这时需要在日志中逆流而上找出其原始异常(Root Cause)，才能确定是在类加载过程的哪一阶段出错和具体出错原因。原始异常通常就在`NoClassDefFoundError`**第一次**出现的地方的前面。

# `java.lang.NoClassDefFoundError`
该异常在API文档中的描述为:

> `public class NoClassDefFoundError extends LinkageError`
> 
> Thrown if the Java Virtual Machine or a ClassLoader instance tries to **load in the definition of a class (as part of a normal method call or as part of creating a new instance using the new expression)** and no definition of the class could be found.
> 
> The searched-for class definition existed when the currently executing class was compiled, but the definition can no longer be found.

# 类加载的几个阶段和分别对应的方法
类加载(Class Loading)过程中的几个阶段包括：加载(Loading)、链接(Linking)、初始化(Initialization)。其中链接阶段又可细分为验证(Verification)、准备(Preparation)、解析(Resolution)三个过程。各个阶段的具体描述请参考周志明或[The Java Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf)，它们分别对应的主要方法简介如下。在遇到类加载相关问题时，可以用[BTrace](https://github.com/btraceio/btrace)的[`@OnMethod`](https://github.com/btraceio/btrace/wiki/BTrace-Annotations#onmethod)注解来跟踪方法执行。

1. 加载

    `protected final Class<?> defineClass(String name, byte[] b, int off, int len, ProtectionDomain protectionDomain)`
    
    > It converts an array of bytes into an instance of class `Class`.

    位于`java.lang.ClassLoader`中。

2. 链接

    `protected final void resolveClass(Class<?> c)`
    
    > Links the specified class. This (misleadingly named) method may be used by a class loader to link a class.
    
    位于`java.lang.ClassLoader`中。

3. 初始化

    `<clinit>()`
    
    类加载过程的最后一步，执行类构造器方法，它是编译器自动收集类中的所有类变量(static)的赋值动作和静态语句块(static {})中的语句合并产生的。

# JVM记录类加载状态
