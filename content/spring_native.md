+++
title = "尝试spring native"
date = "2022-04-26"
[taxonomies]
tags = ["springboot", "graalvm"]
categories = ["Java"]
+++

## 安装GraalVM

由于目前手头上用的是M1的mac，正式版本的GraalVM目前还不支持，只能手工去下载和安装开发版的。
具体版本可以在：https://github.com/graalvm/graalvm-ce-dev-builds/releases 页面中查找，
我下载了“graalvm-ce-java17-darwin-aarch64-dev.tar.gz”这个版本，既基于java17的ARM64位架构版本。

安装流程：
```bash
tar vxf graalvm-ce-java17-darwin-aarch64-dev.tar.gz
sudo xattr -r -d com.apple.quarantine graalvm-ce-java17-22.2.0-dev/Contents/Home
export PATH=`pwd`/graalvm-ce-java17-22.2.0-dev/Contents/Home/bin:$PATH
export JAVA_HOME=`pwd`/graalvm-ce-java17-22.2.0-dev/Contents/Home
```

确认环境变量生效之后，可以查看下`java`命令的输出：
```
❯ java -version
openjdk version "17.0.3" 2022-04-19
OpenJDK Runtime Environment GraalVM CE 22.2.0-dev (build 17.0.3+4-jvmci-22.1-b03)
OpenJDK 64-Bit Server VM GraalVM CE 22.2.0-dev (build 17.0.3+4-jvmci-22.1-b03, mixed mode, sharing)
```

这样就已经使用了刚下载的GraalVM了。

然后安装native-image:
```
gu install native-image
hash -r
```

确认下`native-image`版本：
```
❯ native-image --version
GraalVM 22.2.0-dev Java 17 CE (Java Version 17.0.3+4-jvmci-22.1-b03)
```

## 一个简单的springboot工程
去 http://start.spring.io/ 网站生成一个仅包含“Spring Native”和“Spring Reactive Web”的spring boot工程，下周之后解压缩。

直接尝试构建：

```
./mvnw package -DskipTests -Pnative
```
执行完成之后，会在target下面生成一个和工程名字同名的可执行程序。执行过程中可能会有各种警告和异常，目前还不知道是否会有影响。

```
❯ file target/demo
target/demo: Mach-O 64-bit executable arm64
```

可以看见这个名为demo文件，是一个ARM64架构的可执行程序。直接执行这个可执行程序，可以看见整个spring启动时间大大缩短：

```
2022-04-26 10:51:04.037  INFO 45550 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 0.146 seconds (JVM running for 0.15)
```

如果直接用java命令执行target目录下的jar包`java -jar target/demo-0.0.1-SNAPSHOT.jar`，耗时为：

```
2022-04-26 10:52:29.553  INFO 45636 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 0.908 seconds (JVM running for 1.199)
```

可见借助于GraalVM的AOT能力构建成native文件之后，java应用的启动时间能够大大的提高。

在GraalVM刚出来的时候，也尝试过通过`native-image`来构建本地可执行程序，但是当时底层依赖的netty一直没法通过编译。
随着GraalVM本身的不断优化，以及spring native（和一些相关构建组件）对整个流程的封装，将一个spring应用直接构建成本地可执行程序。

当然，直接构建成可执行程序的最大问题，就是跨平台执行问题。许多类似的语言都提供了交叉编译的方式，但是`native-image`目前还没有
交叉编译支持。在GitHub上找到了一个相关的[issue](https://github.com/oracle/graal/issues/407)，里面讨论了各种困难，
也有一些利用多架构Docker镜像来构建的方案，但是目前还无法实现直接构建出基于另一个平台的二进制可执行文件。
