+++
title = "JVMTI那些事——加载"
date = "2018-07-02"
[taxonomies]
tags = ["jvmti"]
categories = ["Java"]
+++

## JVMTI简介
JVMTI（JVMTM Tool Interface），是JVM对外提供的一系列接口，提供了查看JVM内部状态和控制JVM执行等功能，包括不限于：性能调优、调试、监控、线程分析和覆盖率分析等。

JVMTI提供的接口大致分为2种类型：直接函数调用和异步事件回调。这些函数执行和JVM在同一个进程中，因此可以高性能的运行。注意：JVMTI接口用C/C++的语言提供，最终以动态链接库的形式由JVM加载并运行。下文对基于JVMTI编写的动态链接库简称为agent。

JVMTI的具体描述和接口，可以参照[oracle官方文档](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html)。

## JVMTI agent加载

JVMTI agent加载机制和javaagent类似，事实上，javaagent的加载本身就是依赖JVMTI的。因此agent加载也可以通过两种模式：

1. JVM启动参数
2. JVM启动后动态加载

本文会着重介绍第二种加载的方式的原理。

### JVM启动参数加载

JVM启动命令行支持两种方式加载：

1. `-agentlib:<agent-lib-name>=<options>` 该参数下，只需要提供agent动态链接库的名字。注意：这里的agent-lib-name **不是** 动态链接库的全称，而是去除了动态链接库标准前缀和后缀之后的名称。例如参数为`-agentlib:demo`，对于类UNIX系统来说，JVM会加载`libdemo.so`，而对于windows则会加载`demo.dll`。同时，由于此处没有指定动态链接库的路径，JVM的查找路径和JNI库查找路径相同，对于类UNIX系统会在`LD_LIBRARY_PATH`环境变量指定的路径下查找，windows则在`PATH`环境变量指定的路径下查找。因此这种方式不太适合工程化部署。
2. `-agentpath:<path-to-agent>=<options>` 该参数下，需要提供agent的全路径。例如`-agentpath:/tmp/libdemo.so`。这种方式下，JVM不会试图扩展动态链接库的前缀和后缀。


### JVM启动后动态加载

对于大多数情况，agent都需要在JVM启动后动态加载，因此本文着重介绍这种加载方式。注意，本文仅包括Linux平台，其他平台可能存在不同的加载方式。

加载agent的最简单方式是通过`com.sun.tools.attach.VirtualMachine`类来进行。该类由jdk的lib目录下的`tools.jar`提供。注意该类默认不在classpath中，因此编译和运行的时候都需要加入到classpath中（具体方式可以google下，maven可以加入系统依赖）。因此我们从这个类的实现入手，看下openJDK在Linux平台下是如何完成动态链接库加载的。

首先`com.sun.tools.attach.VirtualMachine`类是一个抽象类，源码位于`jdk/src/share/classes/com/sun/tools/attach/VirtualMachine.java`，根据其继承关系，最终找到两个核心的类：

* `HotSpotVirtualMachine`：该类是HotSpot虚拟机的实现
* `LinuxVirtualMachine`：该类是HotSpot虚拟机在Linux系统下的实现。

注意`LinuxVirtualMachine`类源码位于`jdk/src/solaris/classes/sun/tools/attach`目录下。

先看看`HotSpotVirtualMachine`中对于attach相关方法的实现：

```java
public void loadAgentLibrary(String agentLibrary, String options)
    throws AgentLoadException, AgentInitializationException, IOException
{
    loadAgentLibrary(agentLibrary, false, options);
}

/*
 * Load agent - absolute path of library provided to target VM
 */
public void loadAgentPath(String agentLibrary, String options)
    throws AgentLoadException, AgentInitializationException, IOException
{
    loadAgentLibrary(agentLibrary, true, options);
}

/*
     * Load JPLIS agent which will load the agent JAR file and invoke
     * the agentmain method.
     */
public void loadAgent(String agent, String options)
    throws AgentLoadException, AgentInitializationException, IOException
{
    String args = agent;
    if (options != null) {
        args = args + "=" + options;
    }
    try {
        loadAgentLibrary("instrument", args);
    }
    ...
}
```

从这里可以看见，所有加载相关的方法，最终都调用到了`loadAgentLibrary`方法。另外值得注意的是，专门提供给Java instrumentation API的loadAgent方法，实际上是记载了`instrument`这个库，对应的文件在`$JAVA_HOME/jre/lib/amd64/libinstrument.so`。

再来看下`loadAgentLibrary`方法：
```java
/*
 * Load agent library
 * If isAbsolute is true then the agent library is the absolute path
 * to the library and thus will not be expanded in the target VM.
 * if isAbsolute is false then the agent library is just a library
 * name and it will be expended in the target VM.
 */
private void loadAgentLibrary(String agentLibrary, boolean isAbsolute, String options)
    throws AgentLoadException, AgentInitializationException, IOException
{
    InputStream in = execute("load",
                             agentLibrary,
                             isAbsolute ? "true" : "false",
                             options);
    try {
        int result = readInt(in);
        if (result != 0) {
            throw new AgentInitializationException("Agent_OnAttach failed", result);
        }
    } finally {
        in.close();

    }
}
```
这里调用了`execute`方法，下发了`load`指令。`execute`方法的实现就在`LinuxVirtualMachine`类中。有兴趣的话，还可以再看看`HotSpot specific methods`注释以下的方法：

```java
// --- HotSpot specific methods ---

// same as SIGQUIT
public void localDataDump() throws IOException {
    executeCommand("datadump").close();
}

// Remote ctrl-break. The output of the ctrl-break actions can
// be read from the input stream.
public InputStream remoteDataDump(Object ... args) throws IOException {
    return executeCommand("threaddump", args);
}

// Remote heap dump. The output (error message) can be read from the
// returned input stream.
public InputStream dumpHeap(Object ... args) throws IOException {
    return executeCommand("dumpheap", args);
}
...
```
有没有很熟悉？的确`jstack`、`jmap`等工具在不使用`-F`参数的时候，就是通过这些方法获取JVM数据的。

最核心的是`execute`方法的实现：（`LinuxVirtualMachine`）

```java
InputStream execute(String cmd, Object ... args) throws AgentLoadException, IOException {
    assert args.length <= 3;                // includes null

    // did we detach?
    String p;
    synchronized (this) {
        if (this.path == null) {
            throw new IOException("Detached from target VM");
        }
        p = this.path;
    }

    // create UNIX socket
    int s = socket();

    // connect to target VM
    try {
        connect(s, p);
    } catch (IOException x) {
        close(s);
        throw x;
    }

    IOException ioe = null;

    // connected - write request
    // <ver> <cmd> <args...>
    try {
        writeString(s, PROTOCOL_VERSION);
        writeString(s, cmd);

        for (int i=0; i<3; i++) {
            if (i < args.length && args[i] != null) {
                writeString(s, (String)args[i]);
            } else {
                writeString(s, "");
            }
        }
    } catch (IOException x) {
        ioe = x;
    }
    ...
```
从上述代码可以看见，在Linux平台上和JVM交互，实际上是通过UNIX socket来进行的。这里的p是在构造方法中已经初始化好的socket文件。默认情况下JVM是不会暴露这个socket文件的，因此在连接之前，还需要一些额外的操作，具体在`LinuxVirtualMachine`类的构造方法中：

```java
LinuxVirtualMachine(AttachProvider provider, String vmid)
    throws AttachNotSupportedException, IOException
{
    super(provider, vmid);

    // This provider only understands pids
    int pid;
    try {
        pid = Integer.parseInt(vmid);
    } catch (NumberFormatException x) {
        throw new AttachNotSupportedException("Invalid process identifier");
    }

    // Find the socket file. If not found then we attempt to start the
    // attach mechanism in the target VM by sending it a QUIT signal.
    // Then we attempt to find the socket file again.
    path = findSocketFile(pid);
    if (path == null) {
        File f = createAttachFile(pid);
        try {
            // On LinuxThreads each thread is a process and we don't have the
            // pid of the VMThread which has SIGQUIT unblocked. To workaround
            // this we get the pid of the "manager thread" that is created
            // by the first call to pthread_create. This is parent of all
            // threads (except the initial thread).
            if (isLinuxThreads) {
                ...
            } else {
                sendQuitTo(pid);
            }

            // give the target VM time to start the attach mechanism
            int i = 0;
            long delay = 200;
            int retries = (int)(attachTimeout() / delay);
            do {
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException x) { }
                path = findSocketFile(pid);
                i++;
            } while (i <= retries && path == null);
            if (path == null) {
                throw new AttachNotSupportedException(
                    "Unable to open socket file: target process not responding " +
                    "or HotSpot VM not loaded");
            }
        } finally {
            f.delete();
        }
    }
    // Check that the file owner/permission to avoid attaching to
    // bogus process
    checkPermissions(path);

    // Check that we can connect to the process
    // - this ensures we throw the permission denied error now rather than
    // later when we attempt to enqueue a command.
    int s = socket();
    try {
        connect(s, path);
    } finally {
        close(s);
    }
}
```
让JVM创建socket文件的方式分为两步，首先是创建attach文件。该文件命名规则为`.attach_pid<pid>`，文件存放位置是待连接JVM的工作目录（从/proc/<PID>/cwd中获取），或者是系统临时目录（对于Linux来说就是/tmp目录）。创建成功之后，再向进程发送SIGQUIT信号，一切正常的话，JVM会在系统临时目录中生成名为`.java_pid<pid>`的socket文件，其他进程可以通过这个进程来进行通信。

当检测到socket文件之后，还会再确认下该文件权限。这个权限比较严格，只有确认该文件创建人和组和当前进程相同，并且文件访问权限中的组用户和其他用户读写权限均为0的时候，才认为有效。即使用root帐号执行jstack，也是无法通过这种方式来和JVM交互的。这个检查是native方法，源码在`jdk/src/solaris/native/sun/tools/attach/LinuxVirtualMachine.c`，大致的判断逻辑是：

```c
if ( (sb.st_uid != uid) || (sb.st_gid != gid) ||
     ((sb.st_mode & (S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH)) != 0) ) {
    JNU_ThrowIOException(env, "well-known file is not secure");
}
```


最后，让我们回到通过socket发送命令的代码，可以看见所有对socket文件的写操作，都是通过`writeString`方法完成的。该方法源码为：

```java
private void writeString(int fd, String s) throws IOException {
    if (s.length() > 0) {
        byte b[];
        try {
            b = s.getBytes("UTF-8");
        } catch (java.io.UnsupportedEncodingException x) {
            throw new InternalError(x);
        }
        LinuxVirtualMachine.write(fd, b, 0, b.length);
    }
    byte b[] = new byte[1];
    b[0] = 0;
    write(fd, b, 0, 1);
}
```
当然了，最终写文件还是通过native方法实现的，这里会将写入的字符串转换成byte数组，然后写入到socket文件，然后再写入一个`\0`来作为结尾。

通过前面`execute`方法源码就可以推导出整个交互协议：`版本号\0命令\0参数1\0参数2\0参数3\0`，当参数不存在的时候，写入空字符串，但是其中的`\0`仍然存在。这里我们以`HotSpotVirtualMachine`方法中的获取系统属性命令作为例子，用简单的shell脚本来模拟整个attach流程：

```java
public Properties getSystemProperties() throws IOException {
    InputStream in = null;
    Properties props = new Properties();
    try {
        in = executeCommand("properties");
        props.load(in);
    } finally {
        if (in != null) in.close();
    }
    return props;
}
```
这个命令没有参数，命令名字为`properties`。从创建socket文件开始，用简单的shell脚本模拟：

```bash
#!/bin/bash

PID=$1
ATTACH_FILE="/tmp/.attach_pid$PID"
SO_FILE="/tmp/.java_pid$PID"

touch $ATTACH_FILE
kill -SIGQUIT $PID

sleep 1
printf '1\0properties\0\0\0\0' | nc -U $SO_FILE | tail -n +2
```

不出意外在控制台就会输出对应JVM的系统属性了。注意输出第一行有个0,这是JVM响应该命令的状态，在脚本里面就直接删除忽略了。把这个脚本改吧改吧，就能不通过tools.jar提供的attach接口，直接将agent注入到目标JVM中了。
