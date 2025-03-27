+++
title = "JVMTI那些事——和Java相互调用"
date = "2019-02-14"
[taxonomies]
tags = ["jvmti"]
categories = ["Java"]
+++

前面几篇文章介绍了JVMTI接口的一些基本概念，以及如何编写一个基于JVMTI的agent。
那些简单的例子只是JVMTI agent自己实现一些简单的功能，如果能够将JVMTI提供的接口经过包装之后提供给Java使用，
能够发挥更大的作用。

## 需求

本文源自一个实际的需求：业务代码需要在不依赖spring容器的情况下，或者spring的ApplicationContext对象实例，
进而能够获取到spring管理的所有bean。对这个需求再进行通用扩展，可以简化成：传入一个类（或者是类签名，这里有个ClassLoader的坑，后面会提到。），返回该类在堆中的所有实例。

## 实现

由于JVMTI只提供了C/C++接口，因此要让业务代码能够调用到，必须要通过JNI接口。因此实现需要分成两个部分：Java部分和C++部分。

其中Java部分比较简单，内部会定义一个native方法来最终调用JNI提供的函数。C++部分也可以拆成两部分，一部分基于JVMTI，
用于处理JVMTI规范定义的初始化和JVMTI API调用的实现；另一部分基于JNI，用于最终向Java部分提供实现。

### Java API

需求章节事实上已经确认了Java端的API，提供的方法定义很简单，分别是：
```java
class ObjectQueryUtils {
    public static List<Object> queryObjects(String name) {
        ...
    }

    public static <T>List<T> queryObjects(Class klass) {
        ...
    }
}
```

这里定义了两个方法，因为第一个方法（参数为String类型的类名称）使用成本比较高，但是可以解决Class对象的ClassLoader隔离问题（下面会提到）。第二个就是通过Class对象直接查询。

对应的native方法也有两个：
```java
synchronized private static native Object[] doQuery(String name);
synchronized private static native Object[] doQuery(Class klass);
```

每个方法对应各自对外提供的接口。这里为了在JNI中构造方便，因此返回值直接使用了Object数组的方式。

这里有两点需要注意：

1. 由于JVMTI实现有非线程安全的操作，因此这里给native方法增加了`synchronized`修饰。
2. native方法的实现基于JVMTI，因此如果采用动态注入的方式提供，这里一定要在调用时catch`Throwable`，或者是对应的`LinkageError`，如果调用时agent还没有注入，光catch`Exception`是没用的。

### JVMTI agent

在前文[JVMTI那些事——c++编写的agent](@/java_c_agent.md)中，已经介绍了基于C++编写Java Agent的结构（基于CMake），这里使用类似的方式。其中主要依赖的JVMTI接口是：[IterateOverInstancesOfClass](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#IterateOverInstancesOfClass)。该接口属于Heap (1.0)时的接口，目前已经不建议使用，但是新版本的接口定义明确说了返回的类实例不包括其子类和实现了该接口的类，因此对于需求中提到的需要查询所有的`ApplicationContext`接口的实例不适用。

在JVMTI调用部分，定义了一个类，完成实际的查询逻辑：

```c++
class ObjectQuery
{
public:
    ObjectQuery(JavaVM* vm);
    ~ObjectQuery();

    jobjectArray query(JNIEnv *env, jstring className);
    jobjectArray query(JNIEnv *env, jclass klass);

private:
    jobjectArray doQuery(JNIEnv *env, std::vector<jclass> classes);

    jvmtiEnv *jvmti;
    jlong tagValue;
};
```

这个类有2个函数，刚好对应2个不同入参的native函数实现。所有实际的查询操作都在`doQuery`函数中实现：

```c++
static jvmtiIterationControl JNICALL heapObjectCallback(jlong class_tag,
    jlong size, jlong* tag_ptr, void* user_data) {
    jlong *tagValue = reinterpret_cast<jlong*>(user_data);
    *tag_ptr = *tagValue;
    DEBUG_LOG << "tag obj by: " << *tag_ptr << "\n";
    return JVMTI_ITERATION_CONTINUE;
}

jobjectArray ObjectQuery::doQuery(JNIEnv *env, std::vector<jclass> classes) {
    if(classes.empty()) {
        return initObjectArray(env, 0);
    }
    jlong *tag = &this->tagValue;
    for(auto klass : classes) {
        jvmti->IterateOverInstancesOfClass(klass, JVMTI_HEAP_OBJECT_EITHER, heapObjectCallback, tag);
    }

    jint count;
    jobject *instances;
    jvmti->GetObjectsWithTags(1, tag, &count, &instances, NULL);

    DEBUG_LOG << "find obj count: " << count << "\n";

    jobjectArray result =initObjectArray(env, count);
    for(int i = 0; i < count; i++) {
        env->SetObjectArrayElement(result, i, instances[i]);
    }

    jvmti->Deallocate(reinterpret_cast<unsigned char*>(instances));
    this->tagValue ++;

    return result;
}
```

功能很简单，将传入的jclass数组遍历进行打标，打标的回调函数为`heapObjectCallback`，打标内容为一个jlong类型的数字。
最后将所有完成打标的对象取出来，通过JNI提供的函数创建一个Java数组，并返回。

对应的JNI函数，可以先通过javah命令创建对应的函数签名，然后复制过来（可以通过在IDEA中定义一个工具来更方便的操作）。
```c
/*
 * Class:     ObjectQueryUtils
 * Method:    doQuery
 * Signature: (Ljava/lang/Class;)[Ljava/lang/Object;
 */
JNIEXPORT jobjectArray JNICALL Java_ObjectQueryUtils_doQuery__Ljava_lang_Class_2
  (JNIEnv *env, jclass thiz, jclass klass) {
    return objectQuery->query(env, klass);
}

/*
 * Class:     ObjectQueryUtils
 * Method:    doQuery
 * Signature: (Ljava/lang/String;)[Ljava/lang/Object;
 */
extern "C"
JNIEXPORT jobjectArray JNICALL Java_ObjectQueryUtils_doQuery
  (JNIEnv *env, jclass thiz, jstring name) {
      return objectQuery->query(env, name);
}
```

最后将代码编译成一个动态链接库即可。使用时，可以将动态链接库放在jvm启动参数中，或者使用各种方式在运行时注入到对应的jvm中。

## 大坑：ClassLoader隔离

前面多次提到了从Java接口到JNI实现，都使用了2套代码，传入Class对象或者函数签名。这时因为会有CLassLoader隔离的问题。

按照Java规范，JNI调用的时候，会和Java环境共用一个ClassLoader，因此如果只提供传入Class对象的接口，会导致只能查询到加载该Class对象ClassLoader对应的实例。事实上，在纯Java环境中，不同ClassLoader加载的Class对象，也是不相同的，这也是很多基于ClassLoader来做类加载隔离的原理。

再回到需求中来，由于业务代码可能会存在由多个ClassLoader加载的Class对象，如果需要用户在调用时确认ClassLoader，很多时候是不现实的。这时候，又可以借助JVMTI了。JVMTI的class相关接口可以列出JVM中加载的所有Class对象（在C环境中为jclass对象）。因此，上层代码中会增加一个传入类签名的接口，该接口的最终实现中，会通过[GetLoadedClasses](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#GetLoadedClasses)函数比对所有加载的类，如果类签名一致，则将该jclass对象用于后续的实例查询。

这里就几个问题需要注意：

1. 在JVMTI中类签名为JVM内部表示方式，即用L开头，使用/分隔包，分号结尾，具体参见[JNI文档](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/types.html#type_signatures)。
2. 由于返回的对象可能由不同的ClassLoader加载，使用时需要特别注意筛选实际需要调用的对象。
