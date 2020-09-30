---
title: cmake.1.install
date: 2020-5-11
tags: [cmake]
categories: 工具
---

# CMAKE介绍

cmake是kitware公司以及一些开源开发者在开发几个工具套件(VTK)的过程中所产生的衍生品。后来经过发展，最终形成体系，在2001年成为一个独立的开放源代码项目。其官方网站是[www.cmake.org](https://cmake.org/)， 可以通过访问官方网站来获得更多关于cmake的信息，而且目前官方的英文文档比以前有了很大的改进，可以作为实践中的参考手册。

cmake的流行离不开KDE4的选择。KDE开发者在使用autotools近10年之后，终于决定为KDE4项目选择一个新的工程构建工具。之所以如此，用KDE开发者们自己话来说，就是：只有少数几个“编译专家”能够掌握KDE现在的构建体系。在经历了unsermake，scons以及cmake的选型和尝试之后，KDE4最终决定使用cmake作为自己的构建系统。在迁移过程中，进展一场的顺利，并获得了cmake开发者的支持。所以，目前的KDE4开发版本已经完全使用cmake来进行构建。

随着cmake 在KDE4项目中的成功，越来越多的项目正在使用cmake作为其构建工具，这也使得cmake正在成为一个主流的构建体系。

# 一、 为何要使用构建系统
决定代码的组织方式及其编译方式，也是程序设计的一部分。因此，我们需要cmake和autotools这样的工具来帮助我们构建并维护项目代码。

看到这里，也许你会想到makefile，makefile不就是管理代码自动化编译的工具吗？为什么还要用别的构建工具？

其实，cmake和autotools正是makefile的上层工具，它们的目的正是为了产生可移植的makefile，并简化自己动手写makefile时的巨大工作量。如果你自己动手写过makefile，你会发现，makefile通常依赖于你当前的编译平台，而且编写makefile的工作量比较大，解决依赖关系时也容易出错。因此，对于大多数项目，应当考虑使用更自动化一些的 cmake或者autotools来生成makefile，而不是上来就动手编写。

总之，项目构建工具能够帮我们在不同平台上更好地组织和管理我们的代码及其编译过程，这是我们使用它的主要原因。

# 二、 cmake主要特点
- 1.开放源代码,使用类 BSD 许可发布。

- 2.跨平台,并可生成 native 编译配置文件,在 Linux/Unix 平台,生成 makefile,在 苹果平台,可以生成 xcode,在 Windows 平台,可以生成 MSVC 的工程文件。

- 3.能够管理大型项目,KDE4 就是最好的证明。

- 4.简化编译构建过程和编译过程。Cmake 的工具链非常简单:cmake+make。
- 5.高效率，按照 KDE 官方说法,CMake 构建 KDE4 的 kdelibs 要比使用 autotools 来 构建 KDE3.5.6 的 kdelibs 快 40%,主要是因为 Cmake 在工具链中没有 libtool。

- 6.可扩展,可以为 cmake 编写特定功能的模块,扩充 cmake 功能。

# 三、 install
ubuntu系统下：
```
sudo apt-get install cmake
```
默认安装在`/usr/share/cmake-xxx`

# 四、cmake 语法
- CMakeLists.txt 的语法比较简单,由命令、注释和空格组成,其中命令是不区分大小写的,符号"#"后面的内容被认为是注释。 
- 命令由命令名称、小括号和参数组成,参数之间使用空格进行间隔
``` shell
PROJECT(main) 
CMAKE_MINIMUM_REQUIRED(VERSION 2.6)
AUX_SOURCE_DIRECTORY(. DIR_SRCS)
ADD_EXECUTABLE(main ${DIR_SRCS})
```
- 解析：
	- 第一行是一条命令,名称是 PROJECT ,参数是 main ,该命令表示项目的名称是 main 。
	- 第二行的命令限定了 CMake 的版本。
	- 第三行使用命令 AUX_SOURCE_DIRECTORY 将当前目录中的源文件名称赋值给变量 DIR_SRCS 
	```
	aux_source_directory(<dir> <variable>)
	# 把参数 <dir> 中所有的源文件名称赋值给参数 <variable>
	```
	- 第四行使用命令 ADD_EXECUTABLE 指示变量 DIR_SRCS 中的源文件需要编译 成一个名称为 main 的可执行文件。
