---
title: bitbake desc
date: 2021-3-8
tags: [bitbake]
categories: Learning
---

# Desc
- OE BitBake 是一个软件组建自动化工具程序，像所有的 build 工具一样（比如 make，ant，jam）控制如何去构建系统并且解决构建依赖。但是又区别于功能单一的工程管理工具(比如make)，bitbake不是基于把依赖写死了的makefile，而是收集和管理大量之间没有依赖关系的描述文件(这里我们称为包的配方)，然后自动按照正确的顺序进行构建。oe代表OpenEmbedded，而openembedded是一些用来交叉编译，安装和打包的metadata(元数据)。
- OpenEmbedded 是一些脚本（shell 和 python 脚本）和数据构成的自动构建系统。脚本实现构建过程，包括下载（fetch）、解包（unpack）、打补丁（patch）、配置（configure）、编译（compile）、安装（install）、打包（package）、staging、做安装包（package_write_ipk）、构建文件系统等。
- OE 编译顺序：
```shell
do_setscene
do_fetch
do_unpack
do_path
do_configure
do_qa_configure
do_compile
do_stage
do_install
do_package
do_populate_staging
do_package_write_deb
do_package_write
do_distribute_sources
do_qa_staging
do_build
do_rebuild
```
    - do_compile 这些函数都是在 OpenEmbedded 的 classes 中定义的，而 bitbake 中并没有对这些进行定义。这说明，bitbake 只是 OE 更底层的一个工具，也就是说，OE 是基于 bitbake 架构来完成的。

- BitBake 根据预先定义的元数据执行任务，这些元数据定义了执行任务所需的变量，执行任务的过程，以及任务之间的依赖关系，它们存储在 recipe(.bb)、append(.bbappend)、configuration(.conf)、include(.inc) 和 class(.bbclass) 文件中。
- BitBake 包含一个抓取器，用于从不同的位置获取源码，例如本地文件、源码控制器(git)、网站等。
- 每一个任务单元的结构通过 recipe 文件描述，描述的信息有依赖关系、源码位置、版本信息、校验和、说明等等。
- BitBake 包含了一个 C/S 的抽象概念，可以通过命令行或者 XML-RPC 使用，拥有多种用户接口。

# Definition
- Recipe `.Recipe` 文件是最基本的元数据文件，每个任务单元对应一个 Recipe 文件，后缀是 .bb ，这种文件为 BitBake 提供的信息包括软件包的基本信息（作者、版本、License等）、依赖关系、源码的位置和获取方法、补丁、配置和编译方法、如何打包和安装。
- Configuration `.Configuration` 文件的后缀是 .conf ，它会在很多地方出现，定义了多种变量，包括硬件架构选项、编译器选项、通用配置选项、用户配置选项。主 Configuration 文件是 bitbake.conf ，以 Yocto 为例，位于 ./poky/meta/conf/bitbake.conf ，其他都在源码树的 conf 目录下。
- Classes `.Class` 文件的后缀是 .bbclass ，它的内容是元数据文件之间的共享信息。BitBake 源码树都源自一个叫做 base.bbclass 的文件，在 Yocto 中位于 ./poky/meta/classes/base.bbclass ，它会被所有的 recipe 和 class 文件自动包含。它包含了标准任务的基本定义，例如获取、解压、配置、编译、安装、打包，有些定义只是框架，内容是空的。
- Layers `.Layer` 被用来分类不同的任务单元。某些任务单元有共同的特性，可以放在一个 Layer 下，方便模块化组织元数据，也方便日后修改。例如要定制一套支持特定硬件的系统，可以把与低层相关的单元放在一个 layer 中，这叫做 Board Support Package(BSP) Layer 。
- Append `.Append` 文件的后缀是 .bbappend ，用于扩展或者覆盖 recipe 文件的信息。BitBake 希望每一个 append 文件都有一个相对应的 recipe 文件，两个文件使用同样的文件名，只是后缀不同，例如 formfactor_0.0.bb 和 formfactor_0.0.bbappend 。命名 append 文件时，可以用百分号（%）来通配 recipe 文件名。例如，一个名为 busybox_1.21.%.bbappend 的 apend 文件可以对应任何名为 busybox_1.21.x.bb 的 recipe 文件进行扩展和覆盖，文件名中的 x 可以为任何字符串，比如 busybox_1.21.1.bb、busybox_1.21.2.bb ... 通常用百分号来通配版本号。

# 基本工作流程
- 运行 BitBake 的主要目的是生成一个东西，例如安装包、内核、链接库、或者一个完整的 Linux 系统启动镜像（包括 bootloader、kernel、根文件系统）。当然，你也可以通过使用 bitbake 命令的某些参数，只执行生成过程中的某个步骤，例如编译、获取或清除数据、或者只返回编译环境的信息。

## 分析基本元数据
- 基本元数据由多个文件组成，包括 bblayers.conf 文件（定义项目所需的 layers）、每个 layer 的 layer.conf 文件、以及 bitbake.conf 文件。数据内容有如下几类：
    - Recipes：特定软件包的详情。
    - Class Data：通用构建信息的抽象总结。
    - Configuration Data：针对特定机器的设置，相当于粘合剂，把所有软件集合到一起。
- 基本元数据都具有全局属性，所有它们对所有的 recipes 都有效。
