---
layout:     post
title:      "在Android中启动执行java程序"
subtitle:   " \"在android平台中运行java可执行程序\""
date:       2018-07-20
author:     "Songze Lee"
header-img: "img/post-bg-2018.jpg"
tags:
    - Java
---
> 在Android中启动执行java程序

# 1. 在Android中启动执行java程序

在pc上源代码Hello.java通过javac编译生成Hello.class,通过java命令启动java虚拟机解析执行.

Hello.java
```java
class Hello {
	public static void main(String args[]) {
		System.out.println("hello java!");
	}
}
```
编译:
```java
$ javac Hello.java
$ java Hello
hello java!
```

在Android平台上虚拟机是goole公司自己设计的Dalvik vm，dex是Android平台上(Dalvik 虚拟机)的可执行文件，
因此需要 Hello.java ---(javac 编译)-->Hello.class ---(dx转换)--> dex格式。

以下提供两种方法实现java程序在android中执行。

## 1.1 方法一

### 1.1.1 编译：
```java
$ javac Hello.java
$ dx --dex --output=Hello.jar Hello.class
$ ls
Hello.class  Hello.jar  Hello.java
```
注意 dx命令需要android工程 . build 和 lunch 配置环境变量后才能找到此命令，能够执行。

### 1.1.2 执行：
```java
C:\Users\lisongze>adb push Z:\Android\frameworks\testing\javatest\Hello.jar /data
C:\Users\lisongze> adb shell
root@8860cp0:/ # dalvikvm -cp /data/Hello.jar Hello
dalvikvm -cp /data/Hello.jar Hello
hello java!
或者:
root@8860cp0:/ #  CLASSPATH=/data/Hello.jar app_process . Hello
 CLASSPATH=/data/Hello.jar app_process . Hello
hello java!
```

## 1.2 方法二

放入Android源码工程中编译成可执行文件,这里参考frameworks/base/cmds/am/Android.mk

### 1.2.1 增加 Android.mk和test文件
test

```java
#!/system/bin/sh
#
# Script to start "am" on the device, which has a very rudimentary
# shell.
#
base=/system
export CLASSPATH=$base/framework/hello.jar
exec app_process $base/bin Hello "$@"
```

Android.mk

```java
# Copyright 2008 The Android Open Source Project
#
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := hello
include $(BUILD_JAVA_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE := javatest
LOCAL_SRC_FILES := test
LOCAL_MODULE_CLASS := EXECUTABLES
LOCAL_MODULE_TAGS := optional
include $(BUILD_PREBUILT)

```

### 1.2.2 编译

```java
lisongze@svr04:~/Andriod/frameworks/testing/javatest$ ls
Android.mk  Hello.java  test
lisongze@svr04:mm -B
```

编译生成以下两个文件: 

out/target/product/mobile/system/framework/hello.jar 

out/target/product/mobile/system/bin/javatest

### 1.2.3 执行

```java
adb push hello.jar /system/framework
adb push javatest /system/bin/

adb shell
root@8860cp0:/ # javatest
javatest
hello java!
或者:
root@8860cp0:/ #  CLASSPATH=/system/framework/hello.jar app_process . Hello
 CLASSPATH=/system/framework/hello.jar app_process . Hello
hello java!

```
