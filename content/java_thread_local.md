+++
title = "java thread local到底存储在哪里？"
date = "2020-06-10"
[taxonomies]
tags = ["thread"]
categories = ["Java"]
+++

## java层面
很多情况下，框架为了透传一些标志位等场景，都会使用到线程变量，既ThreadLocal对象。

从定义来说，这个对象会在每个线程中分配一个区域，独立存储对象。通常来说，我们会把ThreadLocal对象放到类的静态成员变量中，并提供初始值，确保每个线程中都能获取到自己的值，同时线程的整个调用栈中可以共享这个变量。

```
static ThreadLocal<String> currentUser = new ThreadLocal<String>() {
	protected String initialValue() {
		return "";
    }
}
```

如上示例代码，我们可以把当前登录用户放到线程变量中，这样整个调用流程中就可以不用传递这个参数，直接获取了。

那么，这个变量实际存储在哪里呢？

### Thread和ThreadLocal
了解ThreadLocal对象的存储，从Java代码层面还是挺清晰的。Thread对象中有一个成员变量：`threadLocals`，它上面有注释：

```
/* ThreadLocal values pertaining to this thread. This map is maintained
    * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

因此，实际`ThreadLocal`对象中保存的数据，是以Map的形式保存在对应的`Thread`对象中。这个Map的key是ThreadLocal对象的this。

这里要特别注意一点：由于保存在Map中的key实际是一个this对象，而我们通常会把这个变量的定义放在类的静态成员变量中。因此，如果使用反射，需要特别**注意**使用的ClassLoader，因为使用不同ClassLoader最终加载的类可能是隔离的，他们里面定义的静态变量this也会不同，就会导致无法在`ThreadLocalMap`中查到对应的值。

## native代码层面
Java层面ThreadLocal如何存储相关代码很清晰，相关文章也很多。但是，仅仅作为`Thread`对象的一个成员变量，Java是怎么实现线程局部变量功能的呢？

最初的猜测，是不是`Thread`对象通过某个native调用，最终调用了pthread相关函数来进行线程局部变量的存储。但是，查看了`Thread`对象的native方法，并没有类似的接口。

因此，查询的重点，从线程创建开始，当然，这里不关注线程创建本身，只是关注线程变量如何存储。

从Java代码看，创建线程通过start0这个native函数来实现。那么，这里也是排查入口。先来看看对应的JNI入口：

```
#jdk/src/share/native/java/lang/Thread.c
{"start0",           "()V",        (void *)&JVM_StartThread},
```
`Thread`没有使用一对一的方式来注册JNI，而是通过JNI的`RegisterNatives`函数来进行注册。

因此现在目标变成了`JVM_StartThread`函数，实现在`hotspot/src/share/vm/prims/jvm.cpp`文件中。前面各种线程的初始化都暂时忽略，其中创建Java线程的代码为：
```
native_thread = new JavaThread(&thread_entry, sz);
```

这里直接new了一个JavaThread对象，JavaThread定义在`hotspot/src/share/vm/runtime/thread.hpp`文件中。其中有很多个Thread对象，可以参照其注释来了解继承关系：
```
// Class hierarchy
// - Thread
//   - NamedThread
//     - VMThread
//     - ConcurrentGCThread
//     - WorkerThread
//       - GangWorker
//       - GCTaskThread
//   - JavaThread
//   - WatcherThread
```
这里我们重点关注业务线程，也就是`JavaThread`对象，其他线程除了`VMThread`（JVM执行VMOps的线程）和`WatcherThread`（执行周期性监控任务的线程）之外，主要是用来做GC的线程。

回过来继续看Java线程的创建，实现代码在`hotspot/src/share/vm/runtime/thread.cpp`文件中，首先看JavaThread的构造函数：

```
os::create_thread(this, thr_type, stack_sz);
```
实际创建线程的语句是这个。最终要创建线程，当然和操作系统相关了，这里还是只关注Linux系统中的实现。具体文件为：`hotspot/src/os/linux/vm/os_linux.cpp`，具体创建线程，还是通过pthread的函数来实现：
```
int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);
```
其中第三个参数是线程创建之后实际执行的函数，也就是说实际新线程创建之后，逻辑都在这个函数里面。在这个函数里面，我们找到了线程变量相关的线索：
```
ThreadLocalStorage::set_thread(thread);
```

这个方法的实现在`hotspot/src/share/vm/runtime/threadLocalStorage.cpp`这个文件，
```
void ThreadLocalStorage::set_thread(Thread* thread) {
  pd_set_thread(thread);

  // The following ensure that any optimization tricks we have tried
  // did not backfire on us:
  guarantee(get_thread()      == thread, "must be the same thread, quickly");
  guarantee(get_thread_slow() == thread, "must be the same thread, slowly");
}
```

好吧，还是和平台相关，源码针对每种CPU架构都不一样，针对AMD64的代码为(hotspot/src/os_cpu/linux_x86/vm/threadLS_linux_x86.cpp)：
```
void ThreadLocalStorage::pd_set_thread(Thread* thread) {
  os::thread_local_storage_at_put(ThreadLocalStorage::thread_index(), thread);
}
```

这里通过`thread_local_storage_at_put`函数将当前线程对象放到线程局部变量中，代码也非常简单，直接通过pthread函数`pthread_setspecific`完成：
```
void os::thread_local_storage_at_put(int index, void* value) {
  int rslt = pthread_setspecific((pthread_key_t)index, value);
  assert(rslt == 0, "pthread_setspecific failed");
}
```

注意下，这里的index是`ThreadLocalStorage`对象在初始化的时候创建的，也是平台相关的，对于Linux来说，直接通过pthread的`pthread_key_create`函数完成：

```
int os::allocate_thread_local_storage() {
  pthread_key_t key;
  int rslt = pthread_key_create(&key, NULL);
  assert(rslt == 0, "cannot allocate thread local storage");
  return (int)key;
}
```

因此，实际Java把线程对象整个放到了线程上下文中。

## 总结
最初想了解JVM在系统层面如何保存ThreadLocal，是为了能够在JVMTI agent的C++代码中能够获取到ThreadLocal。

参照本文，实际Java层的`ThreadLocal`对象本身存储在`Thread`对象中，而Thread对应在C++代码中的实现`JavaThread`，又通过`ThreadLocalStorage`对象，将自己保存到了实际操作系统线程的线程变量中。
