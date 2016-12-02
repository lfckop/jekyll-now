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
> Thrown if the Java Virtual Machine or a ClassLoader instance tries to **load in the definition of a class (as part of a normal method call or as part of creating a new instance using the new expression) and no definition of the class could be found.**
> 
> The searched-for class definition existed when the currently executing class was compiled, but the definition can no longer be found.

# 类加载的几个阶段和分别对应的方法
类加载(Class Loading)过程中的几个阶段包括：加载(Loading)、链接(Linking)、初始化(Initialization)，其中链接阶段又可细分为验证(Verification)、准备(Preparation)、解析(Resolution)三个过程。各个阶段的具体描述请参考周志明或[The Java Virtual Machine Specification - Loading, Linking, and Initializing](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html)，它们分别对应的主要方法简介如下。在遇到类加载相关问题时，可以用[BTrace](https://github.com/btraceio/btrace)的[`@OnMethod`](https://github.com/btraceio/btrace/wiki/BTrace-Annotations#onmethod)注解来跟踪方法执行。

1. **加载**

    `protected final Class<?> defineClass(String name, byte[] b, int off, int len, ProtectionDomain protectionDomain)`
    
    > It converts an array of bytes into an instance of class `Class`.

    位于`java.lang.ClassLoader`中。

2. **链接**

    `protected final void resolveClass(Class<?> c)`
    
    > Links the specified class. This (misleadingly named) method may be used by a class loader to link a class.
    
    位于`java.lang.ClassLoader`中。

3. **初始化**

    `<clinit>()`
    
    `"the class or interface initialization method"`，类加载过程的最后一步，执行**类构造器**方法，它是编译器自动收集类中的所有类变量(`static`)的赋值动作和静态语句块(`static {}`)中的语句合并产生的。

# JVM记录类加载状态

不甚懂*C++*，粗略地翻了下源码。

从[OpenJDK/jdk8/jdk8/hotspot](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509)的源码可以看出虚拟机内部有个类`InstanceKlass`，它有一个`_init_state`变量记录Java类的类加载状态，摘录如下：

[hotspot-87ee5ee27509/src/share/vm/oops/instanceKlass.hpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/instanceKlass.hpp):

> An InstanceKlass is the VM level representation of a Java class. It contains all information needed for a class at execution runtime.

```c++
  enum ClassState {
    allocated,                          // allocated (but not yet linked)
    loaded,                             // loaded and inserted in class hierarchy (but not linked yet)
    linked,                             // successfully linked/verified (but not initialized yet)
    being_initialized,                  // currently running class initializer
    fully_initialized,                  // initialized (successfull final state)
    initialization_error                // error happened during initialization
  };
  
  u1       _init_state;                 // state of class
  
  // initialization state
  bool is_loaded() const                   { return _init_state >= loaded; }
  bool is_linked() const                   { return _init_state >= linked; }
  bool is_initialized() const              { return _init_state == fully_initialized; }
  bool is_not_initialized() const          { return _init_state <  being_initialized; }
  bool is_being_initialized() const        { return _init_state == being_initialized; }
  bool is_in_error_state() const           { return _init_state == initialization_error; }
```

[hotspot-87ee5ee27509/src/share/vm/oops/instanceKlass.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/instanceKlass.cpp):

关注类加载的最后一步：初始化，它由6种**主动引用**来触发（[A class or interface C may be initialized only as a result of](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.5)）。类初始化主要由[`InstanceKlass::initialize_impl`](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/instanceKlass.cpp#l777)函数来完成，它基本按照[JVM Specification - Initialization](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.5)中描述的`1 - 12`步来执行，摘录部分如下：

``` c++
void InstanceKlass::initialize_impl(instanceKlassHandle this_oop, TRAPS) {
  // Make sure klass is linked (verified) before initialization
  // A class could already be verified, since it has been reflected upon.
  this_oop->link_class(CHECK);

    // Step 5
    if (this_oop->is_in_error_state()) {
      DTRACE_CLASSINIT_PROBE_WAIT(erroneous, InstanceKlass::cast(this_oop()), -1,wait);
      ResourceMark rm(THREAD);
      const char* desc = "Could not initialize class ";
      const char* className = this_oop->external_name();
      size_t msglen = strlen(desc) + strlen(className) + 1;
      char* message = NEW_RESOURCE_ARRAY(char, msglen);
      if (NULL == message) {
        // Out of memory: can't create detailed error message
        THROW_MSG(vmSymbols::java_lang_NoClassDefFoundError(), className);
      } else {
        jio_snprintf(message, msglen, "%s%s", desc, className);
        THROW_MSG(vmSymbols::java_lang_NoClassDefFoundError(), message);
      }
    }
      
  // Step 8
  {
    assert(THREAD->is_Java_thread(), "non-JavaThread in initialize_impl");
    JavaThread* jt = (JavaThread*)THREAD;
    DTRACE_CLASSINIT_PROBE_WAIT(clinit, InstanceKlass::cast(this_oop()), -1,wait);
    // Timer includes any side effects of class initialization (resolution,
    // etc), but not recursive entry into call_class_initializer().
    PerfClassTraceTime timer(ClassLoader::perf_class_init_time(),
                             ClassLoader::perf_class_init_selftime(),
                             ClassLoader::perf_classes_inited(),
                             jt->get_thread_stat()->perf_recursion_counts_addr(),
                             jt->get_thread_stat()->perf_timers_addr(),
                             PerfClassTraceTime::CLASS_CLINIT);
    this_oop->call_class_initializer(THREAD);
  }      
```

可见，类执行初始化之前会先检查确保类已经执行过链接过程。

同样，代码`"step 5"`表明，若抛出异常`"java.lang.NoClassDefFoundError: Could not initialize class xxx.package.ClassName"`(尤其注意**"Could not initialize class"**)，则一定表明是执行**类构造器`<clinit>()`**时出错。

当你见到这个异常描述信息时，说明其实是已经执行完`<clinit>()`了，这时应该找出第一次抛异常的地方！因为类第一次初始化失败后，后续使用此类时不会再次初始化，只会抛出以上信息，此时再用BTrace来跟踪其`<clinit>()`也是徒劳的。用Btrace来启动项目是更好的选择，或者在项目刚刚启动时就进行跟踪。

> 5\. If the Class object for C is in an erroneous state, then initialization is not possible. Release LC and throw a **NoClassDefFoundError**.

代码`"step 8"`执行类构造器`<clinit>()`，`"the class or interface initialization method"`，即调用`call_class_initializer`函数。


# 案例
