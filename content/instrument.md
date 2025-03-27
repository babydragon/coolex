+++
title = "插桩？其实JVM做了很多"
date = "2018-07-03"
[taxonomies]
tags = ["jvmti", "javaagent"]
categories = ["Java"]
+++

## 入口

对JVM中的字节码进行替换，这是JVM通过jvmti接口对外提供的扩展功能。如果要通过Java语言来实现（jvmti提供的是C接口），
可以通过javaagent的方式或者通过tools.jar提供的attach接口进行jar包的加载。直接使用jvmti接口，则可以参照[jvmti文档](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html)编写一个agent让JVM加载。

### javaagent实现

**注意：** 本文仅介绍在线插装方式

javaagent需要以jar包的形式加载，和普通的jar没什么区别，唯一需要在`MANIFEST.MF`文件中增加一些入口。
对于maven工程，可以使用`maven-jar-plugin`来生成该文件内容：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <archive>
            <manifestEntries>
                <Agent-Class>
                    com.cainiao.isimu.instrument.test.inst.InstrumentMain
                </Agent-Class>
                <Can-Redefine-Classes>
                    true
                </Can-Redefine-Classes>
                <Can-Retransform-Classes>
                    true
                </Can-Retransform-Classes>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```
这样会在最终生成的`MANIFEST.MF`中增加`Agent-Class`、`Can-Redefine-Classes`和`Can-Retransform-Classes`这三项，
注意修改`Agent-Class`项的值，设置成agent入口类。

入口类和标准Java入口不一样，需要在类中定义签名为`public static void agentmain(String agentArgs, Instrumentation inst)`的方法。JVM加载该类之后，会执行这个函数。

通过这个入口就可以获取到`Instrumentation`对象了，后面所有的类转换操作都需要通过该对象进行。对于插桩来说，主要是`addTransformer`方法来添加类文件转换器，`retransformClasses`来发起类转换。

具体让JVM动态加载该包的方式，详见[JVMTI那些事——加载](./jvmti_load.m)。

### 直接使用jvmti接口

通过jvmti接口编写基于c/c++语言的agent可以参考：[JVMTI那些事——c++编写的agent](./java_c_agent.md)。
该文没有尝试调用jvmti中的字节码转换函数。

## JVM开始工作
通过jvmti接口提交了类转换请求之后，JVM就需要开始工作了。hotspot对jvmti的实现在`hotspot/src/share/vm/prims/jvmtiEnv.cpp`文件中，对应转换类的函数实现是`jvmtiError
JvmtiEnv::RetransformClasses(jint class_count, const jclass* classes)`。

这个函数前面是各种检查和验证，我们唯一关心的是最后两行：

```C++
VM_RedefineClasses op(class_count, class_definitions, jvmti_class_load_kind_retransform);
VMThread::execute(&op);
```

这里创建了一个名为`VM_RedefineClasses`的vmop，然后通知VMThread进行处理。

### VM_Operation
`VM_Operation`是虚拟机级别的操作，这些操作包含了所有JVM的内置操作，例如GC、获取线程栈等等。
这个类是所有这些操作的基类，该类定义在`hotspot/src/share/vm/runtime/vm_operations.hpp`我们来看下一些默认的操作。

首先，该类定义了`Mode`和`VMOp_Type`两个枚举。第一个表示该操作的模式，第二个表示该操作的类型。事实上所有的类型都在文件开头的宏定义中写明了，这里我们只关心`RedefineClasses`这个类型。`Mode`包括这四种：

```C++
enum Mode {
    _safepoint,       // blocking,        safepoint, vm_op C-heap allocated
    _no_safepoint,    // blocking,     no safepoint, vm_op C-Heap allocated
    _concurrent,      // non-blocking, no safepoint, vm_op C-Heap allocated
    _async_safepoint  // non-blocking,    safepoint, vm_op C-Heap allocated
};
```

根据注释可以知道，`VM_Operation`的模式主要分为阻塞和非阻塞。

主要成员函数有：

```C++
// Called by VM thread - does in turn invoke doit(). Do not override this
void evaluate();

// evaluate() is called by the VMThread and in turn calls doit().
// If the thread invoking VMThread::execute((VM_Operation*) is a JavaThread,
// doit_prologue() is called in that thread before transferring control to
// the VMThread.
// If doit_prologue() returns true the VM operation will proceed, and
// doit_epilogue() will be called by the JavaThread once the VM operation
// completes. If doit_prologue() returns false the VM operation is cancelled.
virtual void doit()                            = 0;
virtual bool doit_prologue()                   { return true; };
virtual void doit_epilogue()                   {}; // Note: Not called if mode is: _concurrent
// Configuration. Override these appropriatly in subclasses.
virtual VMOp_Type type() const = 0;
virtual Mode evaluation_mode() const            { return _safepoint; }
virtual bool allow_nested_vm_operations() const { return false; }
virtual bool is_cheap_allocated() const         { return false; }
virtual void oops_do(OopClosure* f)              { /* do nothing */ };
virtual bool evaluate_at_safepoint() const {
  return evaluation_mode() == _safepoint  ||
         evaluation_mode() == _async_safepoint;
}
virtual bool evaluate_concurrently() const {
  return evaluation_mode() == _concurrent ||
         evaluation_mode() == _async_safepoint;
}
```

注释里面说明了`evaluate`函数不能覆盖，实际业务逻辑必须在`doit`函数中实现。如果需要在实际操作之前有前置操作，可以在`doit_prologue`函数中执行。特别注意的是，`doit_prologue`函数是在Java线程中运行的，这个执行线程实际就是提交`VM_Operation`的线程。

`evaluation_mode`函数返回当前操作的模式，如果子类不覆盖，默认值是阻塞模式。

### VM_RedefineClasses
`VM_RedefineClasses`是`VM_Operation`的子类，实现了类转换的所有逻辑。该类定义和实现分别在`hotspot/src/share/vm/prims/jvmtiRedefineClasses.hpp`和`hotspot/src/share/vm/prims/jvmtiRedefineClasses.cpp`。

对于一个`VM_Operation`的子类，首先需要关心`evaluation_mode`函数。`VM_RedefineClasses`类中找不到该函数，因此它是一个需要在safepoint阻塞的操作。

然后就是核心操作，按照前文说的调用顺序，既`doit_prologue`、`doit`、`doit_epilogue`。

代码比较复杂，我们先从注释上了解每个步骤做了什么：

```C++
// 1) doit_prologue() is called by the JavaThread on the way to a
//    safepoint. It does parameter verification and loads scratch_class
//    which involves:
//    - parsing the incoming class definition using the_class' class
//      loader and security context
//    - linking scratch_class
//    - merging constant pools and rewriting bytecodes as needed
//      for the merged constant pool
//    - verifying the bytecodes in scratch_class
//    - setting up the constant pool cache and rewriting bytecodes
//      as needed to use the cache
//    - finally, scratch_class is compared to the_class to verify
//      that it is a valid replacement class
//    - if everything is good, then scratch_class is saved in an
//      instance field in the VM operation for the doit() call
//
//    Note: A JavaThread must do the above work.
```

在`doit_prologue`阶段，整个操作都是在Java线程中进行的，因此不会阻塞VMThread，也不会被计入safepoint的耗时。
注意整个源码中`the_class`表示待替换的类，`scratch_class`表示新的类。该阶段主要做的就是准备需要的字节码，
如果业务代码中准备新的字节码时间比较长（前面提到的获取新字节码的回调也是在这里发生），这个阶段时间就会变长，但是不会阻塞JVM的核心线程。

然后是最核心的`doit`函数。
```c++
// 2) doit() is called by the VMThread during a safepoint. It installs
//    the new class definition(s) which involves:
//    - retrieving the scratch_class from the instance field in the
//      VM operation
//    - house keeping (flushing breakpoints and caches, deoptimizing
//      dependent compiled code)
//    - replacing parts in the_class with parts from scratch_class
//    - adding weak reference(s) to track the obsolete but interesting
//      parts of the_class
//    - adjusting constant pool caches and vtables in other classes
//      that refer to methods in the_class. These adjustments use the
//      ClassLoaderDataGraph::classes_do() facility which only allows
//      a helper method to be specified. The interesting parameters
//      that we would like to pass to the helper method are saved in
//      static global fields in the VM operation.
//    - telling the SystemDictionary to notice our changes
//
//    Note: the above work must be done by the VMThread to be safe.
```
该函数**必须**在`VMThread`中进行，且调用会从safepoint开始阻塞。该函数的操作主要氛围两个阶段：

* 第一阶段主要是准备工作，执行内容包括：
    - 移除所有断点信息（所以进行过转换的类，需要重新debug，否则是没法断点的）
    - 去优化（Deoptimize）
    - 交换方法和常量池
    - 交换内部类
    - 初始化虚函数表（vtable）和接口函数表（itable）
    - 复制源码相关信息
    - 替换类版本信息
* 第二阶段是依赖处理，主要内容有：
    - 调整所有依赖被替换方法的类的常量池缓存和vtable
    - [JSR-292](https://jcp.org/en/jsr/detail?id=292)支持（invokedynamic）
    - 刷新对象实例缓存，去除废弃的方法
    - 增加被修改类（及其内联类）的`classRedefinedCount`属性

最后的`doit_epilogue`函数比较简单，主要做清理工作和最终统计工作。

## VM_RedefineClasses中的计时器

在看`VM_RedefineClasses`类，我们可以发现有几个定时器相关的字段：

```C++
elapsedTimer  _timer_rsc_phase1;
elapsedTimer  _timer_rsc_phase2;
elapsedTimer  _timer_vm_op_prologue;
```
前两个用于对`doit`函数中的两个阶段分别计时，最后一个为前置操作计时（主要是加载和解析新类时间）。

整个类替换的计时和日志，定义在`jvmtiRedefineClassesTrace.hpp`文件中，里面通过宏来判断是否输出
该阶段的日志。如果需要打印对应的日志，可以在JVM启动的时候添加`-XX:TraceRedefineClasses=XX`来指定。
其中XX参考头文件中的注释，如果要打印所有日志，可以设置为33554431；如果只需要打印计时器时间，可以设置为4。

另外，如果还需要了解整个替换过程中对JVM的开销，还以通过在JVM启动时增加参数`-XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1`。这样JVM会在标准输出中打印所有vmop的时间消耗。注意这个时间仅仅是在VMThread中运行的时间，对于像`VM_RedefineClasses`这种前置有大量时间消耗的，都不会计入其中的vmop时间消耗。
