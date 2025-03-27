+++
title = "java debug初探"
date = "2019-02-20"
[taxonomies]
tags = ["JDWP", "JDI"]
categories = ["Java"]
+++

## JPDA、JDI、JDWP傻傻分不清楚
关于Java debug架构，有一堆相关的名词。其中JPDA是整个debug架构的缩写：Java Platform Debugger Architecture，
整个架构可以从[JPDA文档](https://docs.oracle.com/javase/1.5.0/docs/guide/jpda/architecture.html)最开头了解到：

```
Components                         Debugger Interfaces

               /    |--------------|
              /     |     VM       |
debuggee ----(      |--------------|  <------- JVMTI - Java VM Tool Interface
              \     |   back-end   |
               \    |--------------|
               /           |
comm channel -(            |  <--------------- JDWP - Java Debug Wire Protocol
               \           |
                    |--------------|
                    | front-end    |
                    |--------------|  <------- JDI - Java Debug Interface
                    |      UI      |
                    |--------------|
```

从图中可以看出，

* JDI: Java Debug Interface，作为整个debug架构的客户端API，封装了一些Java API用于debug。包括连接JVM，断点等等功能。
* JDWP：Java Debug Wire Protocol，它实际是一个传输协议，定义了debug客户端和被debug的JVM之间的通信协议。具体协议参见[文档](https://docs.oracle.com/javase/1.5.0/docs/guide/jpda/jdwp-spec.html)。
* JVMTI：前文已经介绍过，这是JVM提供的扩展接口。


## JDI如何工作
JDI定义了一堆接口，用于实现整个Debug功能。JDI接口文档在：https://docs.oracle.com/javase/8/docs/jdk/api/jpda/jdi/index.html 。

JDI中最核心的的类是`VirtualMachine`，提供连接目标机器debug端口，以及一堆支持的指令。注意tools.jar中包含2个同名对象，JDI相关的在jdi包中。

### JDI使用

```java
public class TempAgent {
    static VirtualMachine vm;
    public static void main(String[] args) throws IOException, IllegalConnectorArgumentsException {
        attach();

        //获取对象
        obtainTempTestObj();
    }

    private static void obtainTempTestObj() {
        List<ReferenceType> referenceTypes = vm.classesByName("TempTest");
        for (ReferenceType referenceType : referenceTypes) {
            List<ObjectReference> instances = referenceType.instances(0L);
            for (ObjectReference instance : instances) {
                System.out.println(instance);
            }
        }
    }

    private static void attach() throws IOException, IllegalConnectorArgumentsException {
        // 一、取得连接器
        VirtualMachineManager vmm = Bootstrap.virtualMachineManager();
        List<AttachingConnector> connectors = vmm.attachingConnectors();
        SocketAttachingConnector sac = null;
        for (AttachingConnector ac : connectors) {
            if (ac instanceof SocketAttachingConnector) {
                sac = (SocketAttachingConnector) ac;
                break;
            }
        }
        if (sac == null) {
            System.out.println("JDI error");
            return;
        }

        // 二、连接到远程虚拟器
        Map<String, Argument> arguments = sac.defaultArguments();
        Connector.Argument hostArg = (Connector.Argument) arguments.get("hostname");
        Connector.Argument portArg = (Connector.Argument) arguments.get("port");

        // hostArg.setValue("127.0.0.1");
        portArg.setValue(String.valueOf(8800));

        vm = sac.attach(arguments);
        vm.process();
    }
}
```

这里通过JDI接口，实现了通过debug端口连接到目标JVM，并获取一个类的对象引用。这里我们来看下实际获取对象引用的方式。

### JDI实现

JDI实现代码在 `jdk/src/share/classes/com/sun/tools/jdi/VirtualMachineImpl.java` 类中，
对于本文关注功能（获得对象引用），代码在`jdk/src/share/classes/com/sun/tools/jdi/ReferenceTypeImpl.java`。

其中获取引用的代：
```java
public List<ObjectReference> instances(long maxInstances) {
    if (!vm.canGetInstanceInfo()) {
        throw new UnsupportedOperationException(
            "target does not support getting instances");
    }

    if (maxInstances < 0) {
        throw new IllegalArgumentException("maxInstances is less than zero: "
                                          + maxInstances);
    }
    int intMax = (maxInstances > Integer.MAX_VALUE)?
        Integer.MAX_VALUE: (int)maxInstances;
    // JDWP can't currently handle more than this (in mustang)

    try {
        return Arrays.asList(
            (ObjectReference[])JDWP.ReferenceType.Instances.
                    process(vm, this, intMax).instances);
    } catch (JDWPException exc) {
        throw exc.toJDIException();
    }
}
```

这里可以看见，所有实际操作，都在JDWP这个类中。但是如果直接搜索这个类名，可以发现JDK源码中不包含这个类的源码。
实际这个类是构建时生成的，生成规则在`jdk/make/gensrc/GensrcJDWP.gmk`中：

```makefile
$(JDK_OUTPUTDIR)/gensrc/com/sun/tools/jdi/JDWP.java: $(JDWP_SPEC_FILE)
        $(MKDIR) -p $(@D)
        $(MKDIR) -p $(JDK_OUTPUTDIR)/gensrc_jdwp_headers
        $(RM) $@ $(JDK_OUTPUTDIR)/gensrc_jdwp_headers/JDWPCommands.h
        $(ECHO) $(LOG_INFO) Creating JDWP.java and JDWPCommands.h from jdwp.spec
        $(TOOL_JDWPGEN) $< -jdi $@ -include $(JDK_OUTPUTDIR)/gensrc_jdwp_headers/JDWPCommands.h
```
其中`TOOL_JDWPGEN`变量在`Tools.gmk`文件中定义：

```
TOOL_JDWPGEN = $(JAVA) -cp $(JDK_OUTPUTDIR)/btclasses build.tools.jdwpgen.Main
```

实际会通过JDWP协议和目标JVM交互。

### 目标JVM实现

每个JDI接口，基本上都有一个对应的实现。所有JDWP后端实现都在`jdk/src/share/back`目录中。

对于获取引用的功能，实现在`ReferenceTypeImpl.c`文件中：

```c
static jboolean
instances(PacketInputStream *in, PacketOutputStream *out)
{
    jint maxInstances;
    jclass clazz;
    JNIEnv *env;

    if (gdata->vmDead) {
        outStream_setError(out, JDWP_ERROR(VM_DEAD));
        return JNI_TRUE;
    }

    env = getEnv();
    clazz = inStream_readClassRef(env, in);
    maxInstances = inStream_readInt(in);
    if (inStream_error(in)) {
        return JNI_TRUE;
    }

    WITH_LOCAL_REFS(env, 1) {
        jvmtiError   error;
        ObjectBatch  batch;

        error = classInstances(clazz, &batch, maxInstances);
        if (error != JVMTI_ERROR_NONE) {
            outStream_setError(out, map2jdwpError(error));
        } else {
            int kk;
            jbyte typeKey;

            (void)outStream_writeInt(out, batch.count);
            if (batch.count > 0) {
                /*
                 * They are all instances of this class and will all have
                 * the same typeKey, so just compute it once.
                 */
                typeKey = specificTypeKey(env, batch.objects[0]);

                for (kk = 0; kk < batch.count; kk++) {
                  jobject inst;

                  inst = batch.objects[kk];
                  (void)outStream_writeByte(out, typeKey);
                  (void)outStream_writeObjectRef(env, out, inst);
                }
            }
            jvmtiDeallocate(batch.objects);
        }
    } END_WITH_LOCAL_REFS(env);

    return JNI_TRUE;
}
```

这里最核心的函数调用是`classInstances`，这个函数在`util.c`文件中。

```c
/* Get instances for one class */
jvmtiError
classInstances(jclass klass, ObjectBatch *instances, int maxInstances)
{
    ClassInstancesData data;
    jvmtiHeapCallbacks heap_callbacks;
    jvmtiError         error;
    jvmtiEnv          *jvmti;

    /* Check interface assumptions */

    if (klass == NULL) {
        return AGENT_ERROR_INVALID_OBJECT;
    }

    if ( maxInstances < 0 || instances == NULL) {
        return AGENT_ERROR_ILLEGAL_ARGUMENT;
    }

    /* Initialize return information */
    instances->count   = 0;
    instances->objects = NULL;

    /* Get jvmti environment to use */
    jvmti = getSpecialJvmti();
    if ( jvmti == NULL ) {
        return AGENT_ERROR_INTERNAL;
    }

    /* Setup data to passed around the callbacks */
    data.instCount    = 0;
    data.maxInstances = maxInstances;
    data.objTag       = (jlong)1;
    data.error        = JVMTI_ERROR_NONE;

    /* Clear out callbacks structure */
    (void)memset(&heap_callbacks,0,sizeof(heap_callbacks));

    /* Set the callbacks we want */
    heap_callbacks.heap_reference_callback = &cbObjectTagInstance;

    /* Follow references, no initiating object, just this class, all objects */
    error = JVMTI_FUNC_PTR(jvmti,FollowReferences)
                 (jvmti, 0, klass, NULL, &heap_callbacks, &data);
    if ( error == JVMTI_ERROR_NONE ) {
        error = data.error;
    }

    /* Get all the instances now that they are tagged */
    if ( error == JVMTI_ERROR_NONE ) {
        error = JVMTI_FUNC_PTR(jvmti,GetObjectsWithTags)
                      (jvmti, 1, &(data.objTag), &(instances->count),
                       &(instances->objects), NULL);
        /* Verify we got the count we expected */
        if ( data.instCount != instances->count ) {
            error = AGENT_ERROR_INTERNAL;
        }
    }
    /* Dispose of any special jvmti environment */
    (void)JVMTI_FUNC_PTR(jvmti,DisposeEnvironment)(jvmti);
    return error;
}
```

除了开头一些准备工作，这里实际的调用使用了jvmti的`FollowReferences`和`GetObjectsWithTags`，
两个函数。第一个函数用于在堆中标记期望的对象，第二个函数从堆中将所有做了标记的对象取出来。

因此，实际上JDWP后端，最终通过jvmti实现了一个agent，通过jvmti的API对外提供服务。
