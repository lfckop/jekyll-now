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

> An **InstanceKlass** is the VM level representation of a Java class. It contains all information needed for a class at execution runtime.

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

同样，代码`"step 5"`表明，若抛出异常`"java.lang.NoClassDefFoundError: Could not initialize class xxx.package.ClassName"`(尤其注意**"Could not initialize class"**)，则表明一定是执行**类构造器`<clinit>()`**时出错。

当你见到这个异常描述信息时，说明其实是已经执行完`<clinit>()`了，这时应该找出第一次抛异常的地方！因为类第一次初始化失败后，后续使用此类时不会再次初始化，只会抛出以上信息，此时再用BTrace来跟踪其`<clinit>()`也是徒劳的。用Btrace来启动项目是更好的选择，或者在项目刚刚启动时就进行跟踪。

> 5\. If the Class object for C is in an erroneous state, then initialization is not possible. Release LC and throw a **NoClassDefFoundError**.

代码`"step 8"`执行类构造器`<clinit>()`，JVM表现为调用`this_oop->call_class_initializer(THREAD)`函数。如果类构造器执行成功，则置类加载状态为成功，即：`this_oop->set_initialization_state_and_notify(fully_initialized, CHECK);`。

如果类构造器执行失败（记抛出的异常为**`E`**），则置类加载状态为初始化错误，即：` this_oop->set_initialization_state_and_notify(initialization_error, THREAD);`；若**`E`**不是`Error`类或其子类，则用**`E`**构造一个`ExceptionInInitializerError`，并抛出；这种情形下，当以后再有对该类的主动引用触发类初始化时，则直接走到`"step 5"`并抛出异常`"java.lang.NoClassDefFoundError: Could not initialize class xxx.package.ClassName"`。


# 案例

## 案例1

有一个spring-boot的项目，包含service、api、web三个子工程，分别被打包成可运行jar并以`java -jar`的方式来启动，它们在测试环境中部署在一台机器上。当web工程频繁更新时，会重新打包整个工程并重新启动web，这样service、api的目标可运行jar包会在运行中被改变。然后，service、api工程会频繁抛出这样的异常：`java.lang.NoClassDefFoundError: xxx/package/ClassName1`，在日志中其原始异常基本是这样的：

```
(java.util.zip.ZipException: invalid code lengths set)
(java.util.zip.ZipException: invalid distance too far back)
java.util.zip.ZipException: invalid block type
Wrapped by: java.lang.ClassNotFoundException: xxx.package.ClassName1
Wrapped by: java.lang.NoClassDefFoundError: xxx/package/ClassName1
Wrapped by: org.springframework.web.util.NestedServletException: Handler processing failed; nested exception is java.lang.NoClassDefFoundError: xxx/package/ClassName1
...
```

其原始异常`java.util.zip.ZipException`的错误描述有好几种，如示列在异常栈最上面的`()`中。

为什么重新打包会导致`ZipException`并导致`NoClassDefFoundError`类加载错误呢？这与spring-boot的类加载策略有关。对boot这种可运行jar(jar in jar，又叫nested jar，或fat jar)，spring-boot会首先计算各个类文件在jar中的位置索引，当需要加载某个类时，就依照位置索引去读二进流解压进行类加载；然而当目标jar被重新打包后，之前计算出的位置索引就不准确了，导致后续的类加载报`java.util.zip.ZipException`异常，并导致类加载失败。参考spring开发者[philwebb在该Issue下的回答](https://github.com/spring-projects/spring-boot/issues/1106)。

    
## 案例2

某工程的某个类由于执行类构造器`<clinit>()`失败而导致类加载失败。其原始异常大致如下：

```
严重: Servlet.service() for servlet [appServlet] in context with path [/saas-ms] threw exception [Handler processing failed; nested exception is java.lang.ExceptionInInitializerError] with root cause
java.util.MissingResourceException: Can't find bundle for base name messages, locale zh_CN
at java.util.ResourceBundle.throwMissingResourceException(ResourceBundle.java:1564)
at java.util.ResourceBundle.getBundleImpl(ResourceBundle.java:1387)
at java.util.ResourceBundle.getBundle(ResourceBundle.java:845)
at com.letvcloud.saas.platform.util.ResourceUtils.getValue(ResourceUtils.java:14)
at com.letvcloud.saas.platform.enums.DaysUnit.<clinit>(DaysUnit.java:14)
...
```
    
类加载失败后，后续对该类的调用都抛出`NoClassDefFoundError`异常：
    
```
严重: Servlet.service() for servlet [appServlet] in context with path [/saas-ms] threw exception [Handler processing failed; nested exception is java.lang.NoClassDefFoundError: Could not initialize class com.letvcloud.saas.platform.enums.DaysUnit] with root cause
java.lang.NoClassDefFoundError: Could not initialize class com.letvcloud.saas.platform.enums.DaysUnit
at com.letv.saas.ms.controller.InviteCodeController.inviteCodeApply(InviteCodeController.java:91)
...
```
