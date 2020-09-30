---
title: cmake.5.demos
date: 2020-5-15
tags: [cmake]
categories: 工具
---

# Test1. 基本
源文件目录树：
```
./test1
  | main.c
  | CMakeLists.txt
```
CMakeLists.txt 内容：
```
CMAKE_MINIMUM_REQUIRED(VERSION 2.6)
PROJECT(HELLO)
AUX_SOURCE_DIRECTORY(. SRC_LIST)
ADD_EXECUTABLE(hello ${SRC_LIST})
```

# Test2. `src`目录文件编译成链接库

源文件目录树：
```
./test2
  | main.c
  | CMakeLists.txt
  | src
    | test2.h
    | test2.c
    | CMakeLists.txt
```
/CMakeLists.txt内容：
```
PROJECT(main)
CMAKE_MINIMUM_REQUIRED(VERSION 2.6)
ADD_SUBDIRECTORY(src)
AUX_SOURCE_DIRECTORY(. DIR_SRCS)
ADD_EXECUTABLE(main ${DIR_SRCS})
TARGET_LINK_LIBRARIES(main Test)
```
/src/CMakeLists.txt内容：
```
AUX_SOURCE_DIRECTORY(. DIR_TEST1_SRCS)
ADD_LIBRARY(Test ${DIR_TEST1_SRCS})
```

# Test3. 工程中查找并使用其他程序库的方法
> 在开发软件的时候我们会用到一些函数库,这些函数库在不同的系统中安装的位置可能不同,编译的时候需要首先找到这些软件包的头文件以及链接库所在的目录以便生成编译选项。

工程中假设仅一个源文件`main.c`，但会到调用到其他.h libdb_xxx.so


[引用自：blog.csdn.net/kai_zone/article/details/82656964](https://blog.csdn.net/kai_zone/article/details/82656964?_blank)
