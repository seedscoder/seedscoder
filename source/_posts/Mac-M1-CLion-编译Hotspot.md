---
title: Mac M1 CLion 编译 Hotspot
date: 2024-01-11 23:35:15
tags:
    - 技术
    - Hotspot
categories: 
    - JVM
    - 编译
---


### 环境信息
- 系统：Mac M1 14.2.1 

- xcode 版本： 
  - xcodebuild -version


```
Xcode 15.0.1
Build version 15A507
```





### 准备代码

从 GitHub 将 jdk 源码 clion 克隆至本地 (  `$JDK_SOURCE_CODE_DIR` )

```
git clone https://github.com/openjdk/jdk.git
```





### 编译 jdk 源码



- 进入 ` $JDK_SOURCE_CODE_DIR/jdk` ，执行一下命令

> configure --disable-warnings-as-errors --with-debug-level=slowdebug --with-jvm-variants=server



控制台报错提示：`configure`  没有执行权限，运行以下命令即可，修改 `configure`  文件对应权限之后，再次运行编译命令

> chmod +x configure



> configure --disable-warnings-as-errors --with-debug-level=slowdebug --with-jvm-variants=server



控制台报错提示：缺少 `autoconf`  ，此时执行控制台提示的安装命令即可，安装成功之后，再次执行命令，成功。

> brew install autoconf



> ./configure --disable-warnings-as-errors --with-debug-level=slowdebug --with-jvm-variants=server



```
* C++ Compiler:   Version 15.0.0 (at /usr/bin/clang++ -std=gnu++11)

Build performance summary:
* Build jobs:     10
* Memory limit:   32768 MB
```



> make images

- 出现如下错误

```
Building target 'images' in configuration 'macosx-aarch64-server-slowdebug'
Compiling up to 1 files for BUILD_TOOLS_HOTSPOT
Compiling up to 8 files for BUILD_TOOLS_LANGTOOLS
xattr: [Errno 13] Permission denied: '/Users/me/workspace/open-source/jdk/build/macosx-aarch64-server-slowdebug/jdk/conf/management/jmxremote.password.template'
make[3]: *** [/Users/me/workspace/open-source/jdk/build/macosx-aarch64-server-slowdebug/jdk/conf/management/jmxremote.password.template] Error 1
make[3]: *** Deleting file `/Users/me/workspace/open-source/jdk/build/macosx-aarch64-server-slowdebug/jdk/conf/management/jmxremote.password.template'
make[3]: *** Waiting for unfinished jobs....
make[2]: *** [jdk.management.agent-copy] Error 2
make[2]: *** Waiting for unfinished jobs....

ERROR: Build failed for target 'images' in configuration 'macosx-aarch64-server-slowdebug' (exit code 2)

No indication of failed target found.
HELP: Try searching the build log for '] Error'.
HELP: Run 'make doctor' to diagnose build problems.

make[1]: *** [main] Error 2
make: *** [images] Error 2
```



- 解决办法：
  - 删除  ` $JDK_SOURCE_CODE_DIR/jdk/build` 目录。换用系统自带的 `terminal`，再次执行 `make images`

> https://bugs.openjdk.org/browse/JDK-8299148?page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel&showAll=true



编译成功之后的会出现如下内容：



```
Creating support/demos/image/jfc/Font2DTest/Font2DTest.jar
Creating support/demos/image/jfc/J2Ddemo/J2Ddemo.jar
Creating support/demos/image/jfc/SwingSet2/SwingSet2.jar
Creating support/demos/image/jfc/Metalworks/Metalworks.jar
Creating support/demos/image/jfc/Notepad/Notepad.jar
Creating support/demos/image/jfc/Stylepad/Stylepad.jar
Creating support/demos/image/jfc/SampleTree/SampleTree.jar
Creating support/demos/image/jfc/TableExample/TableExample.jar
Creating support/demos/image/jfc/TransparentRuler/TransparentRuler.jar
Compiling up to 2 files for CLASSLIST_JAR
Creating support/classlist.jar
Creating jdk.jlink.jmod
Creating java.base.jmod
Creating jdk image
Creating CDS archive for jdk image for server
Creating CDS-NOCOOPS archive for jdk image for server
Stopping javac server
Finished building target 'images' in configuration 'macosx-aarch64-server-slowdebug'
```





### 验证

进入 ` $JDK_SOURCE_CODE_DIR/jdk/build/macosx-aarch64-server-slowdebug/jdk/bin` 目录，执行以下命令

> ./java -version

输出如下内容：

```
openjdk version "23-internal" 2024-09-17
OpenJDK Runtime Environment (slowdebug build 23-internal-adhoc.me.jdk)
OpenJDK 64-Bit Server VM (slowdebug build 23-internal-adhoc.me.jdk, mixed mode)
```













