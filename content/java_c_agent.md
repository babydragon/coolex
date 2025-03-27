+++
title = "JVMTI那些事——c++编写的agent"
date = "2018-07-05"
[taxonomies]
tags = ["jvmti"]
categories = ["Java"]
+++

## 简介

前文[介绍](@/jvmti_load.md)过JVMTI的基本知识，事实上JVMTI本身就是JVM通过c语言对外提供的一组接口。
tools.jar中提供的插装接口，最终是通过libinstrument.so这个动态链接库调用JVMTI接口实现的。
因此本文通过一个简单的示例，实现一个监控JVM异常并打印出来的agent。

## 代码结构
使用JVMTI接口需要依赖`jvmti.h`、`jni.h`、`jni_md.h`这几个头文件，当然使用的时候只需要include `jvmti.h`这一个头文件即可。

为了方便管理依赖，这里使用cmake进行依赖管理（实际上只是查找了下jni相关的东西）：

```cmake
#CMakeLists.txt
cmake_minimum_required(VERSION 2.6)
project(javaagent)

find_package(JNI REQUIRED)

include_directories(${JAVA_INCLUDE_PATH} ${JAVA_INCLUDE_PATH2})

add_library(javaagent SHARED main.cpp exception.cpp)
```

在cmake提供的`FindJNI.cmake`文件中，会设置两个头文件相关的环境变量：`JAVA_INCLUDE_PATH`和`JAVA_INCLUDE_PATH2`，
刚好包含前面提到的头文件以及平台相关头文件所在目录。

### 入口

JVMTI文档中介绍过，agent有2个入口函数，分别是启动时和挂载时。这个示例只实现了随JVM启动功能，因此只实现了启动时的入口函数。

```c
#include <jvmti.h>

#include "exception.h"

ExceptionMonitor* monitor;

JNIEXPORT jint JNICALL
Agent_OnLoad(JavaVM *vm, char *options, void *reserved)
{
    monitor = new ExceptionMonitor(vm);
    return JNI_OK;
}

JNIEXPORT void JNICALL
Agent_OnUnload(JavaVM *vm)
{
    delete monitor;
}
```

`Agent_OnLoad`函数会在agent被加载入JVM的时候执行，此时我们就能够获取到jvmti相关上下文。这里简单的将这个上下文传入给
`ExceptionMonitor`类，并初始化。

### JVMTI接口调用

所有JVMTI函数都依赖jvmti上下文，因此需要保存从入口函数传入的jvmti上下文信息。

```c++
#ifndef EXCEPTION_MONITOR_H
#define EXCEPTION_MONITOR_H

#include <jvmti.h>

/**
 * 插装
 */
class ExceptionMonitor
{
public:
    ExceptionMonitor(JavaVM *vm);

private:
    void dealError(jvmtiError err);

private:
    jvmtiEnv *jvmti;
};

#endif // EXCEPTION_MONITOR_H
```
头文件定义了类的定义，因为我们的功能依赖JVMTI的事件回调，因此不需要提供对外接口，只需要在构造函数中完成初始化即可。私有字段只有一个，既jvmti的上下文`jvmtiEnv`指针。

实现也比较简单，实现一个JVMTI的agent，主要有3步：

1. 获取jvmtiEnv对象
2. 添加对应的capability
3. 调用对应的函数

对于事件回调，需要再多一个过程，就是注册回调函数并开启事件监听。

这些流程都在构造函数中一次性完成了：
```c++
ExceptionMonitor::ExceptionMonitor(JavaVM* vm)
{
    jint ret = vm->GetEnv(reinterpret_cast<void**>(&jvmti), JVMTI_VERSION_1_2);

    if(ret != JNI_OK) {
        throw std::invalid_argument("init jvmti env fail");
    }

    // capbilities
    jvmtiCapabilities caps;
    std::memset(&caps, 0, sizeof(caps));
    caps.can_generate_exception_events = 1;
    auto err = jvmti->AddCapabilities(&caps);
    if(err != JVMTI_ERROR_NONE) {
        dealError(err);
        throw std::runtime_error("fail to add can_generate_exception_events capability");
    }

    // callback
    jvmtiEventCallbacks cb;
    cb.Exception = &deal_exception;

    jvmti->SetEventCallbacks(&cb, sizeof(cb));
    //enable
    jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_EXCEPTION,NULL);
}
```

首先通过传入的`JavaVM`对象，获取`jvmtiEnv`对象。然后设置capability，最后设置回调，并启用对应的事件。

对应的回调函数：

```c++
void JNICALL
deal_exception(jvmtiEnv *jvmti_env,
            JNIEnv* jni_env,
            jthread thread,
            jmethodID method,
            jlocation location,
            jobject exception,
            jmethodID catch_method,
            jlocation catch_location)
{
    char *classSign, *classGenericSign;
    auto exceptionClass = jni_env->GetObjectClass(exception);
    jvmti_env->GetClassSignature(exceptionClass, &classSign, &classGenericSign);
    std::cout << "Get exception: " << classSign << "\n";

    char *methodName, *methodSign, *methodGenericSign;
    jvmti_env->GetMethodName(method, &methodName, &methodSign, &methodGenericSign);
    std::cout << " in method: " << methodName << methodSign << "\n";

    jvmtiThreadInfo threadInfo;
    jvmti_env->GetThreadInfo(thread, &threadInfo);
    std::cout << "in Thread: " << threadInfo.name << "\n";

    std::cout << std::endl;

    jvmti_env->Deallocate(reinterpret_cast<unsigned char*>(classSign));
    jvmti_env->Deallocate(reinterpret_cast<unsigned char*>(classGenericSign));
    jvmti_env->Deallocate(reinterpret_cast<unsigned char*>(methodName));
    jvmti_env->Deallocate(reinterpret_cast<unsigned char*>(methodSign));
    jvmti_env->Deallocate(reinterpret_cast<unsigned char*>(methodGenericSign));
    jvmti_env->Deallocate(reinterpret_cast<unsigned char*>(threadInfo.name));
}
```

这里是当JVM出现任何异常时都会进行的回调。首先我们可以通过回调参数`jthread`获取异常产生的线程信息。
通过`jmethodID`获取方法信息，通过`jobject`对象获取实际抛出的异常类型。

前几个数据都有JVMTI函数可以调用，注意最后一个获取类的信息，是通过JNI接口获取的，因此相关函数在JVMTI文档中是查不到的。

## 运行

运行前首先需要构建出动态链接库，这个用cmake就很方便了，直接安装标准构建即可：
```bash
mkdir build
cd build
cmake ..
make
```
构建完成后在build目录中会有`libjavaagent.so`文件，这个就是我们需要的动态链接库了。在java启动的时候增加这个参数：

```
-agentpath:PATH_TO_BUILD_DIR/libjavaagent.so
```
启动之后就可以从标准输出看见JVM产生的所有异常。例如：

```
Get exception: Ljava/lang/Exception;
 in method: genException()Ljava/lang/String;
in Thread: http-nio-7001-exec-5
```

当然，实际JVM会产生超级多异常，可以在打印之前过滤下，或者订阅事件的时候指定线程即可。
