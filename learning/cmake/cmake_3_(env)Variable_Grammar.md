---
title: cmake.3.variable_Env_Grammar
date: 2020-5-13
tags: [cmake]
categories: 工具
---

# 一、 变量引用方式
通常使用`${}`进行变量引用；IF等语句中直接使用变量名，不需要通过`${}`。

# 二、 自定义变量
> 主要有隐式定义和显式定义

### 隐式定义
- PROJECT 指令，会隐式的定义&lt;projectname&gt;_BINARY_DIR 和&lt;projectname&gt;_SOURCE_DIR 两个变量。

### 显式定义
- 使用 SET 指令，就可以构建一个自定义变量了。
- SET(HELLO_SRC main.SOURCE_PATHc),就PROJECT_BINARY_DIR 可以通过${HELLO_SRC}来引用这个自定义变量了.

# 三、 常用变量

### 1. CMAKE_BINARY_DIR
- PROJECT_BINARY_DIR
- &lt;projectname&gt;_BINARY_DIR
- 这三个变量指代的内容是一致的，如果是 in source 编译，指的就是工程顶层目录，如果是 out-of-source 编译,指的是工程编译发生的目录。PROJECT_BINARY_DIR 跟其他指令稍有区别,现在,你可以理解为他们是一致的。

### 2. CMAKE_SOURCE_DIR
- PROJECT_SOURCE_DIR
- &lt;projectname&gt;_SOURCE_DIR
- 这三个变量指代的内容是一致的,不论采用何种编译方式,都是工程顶层目录。
也就是在 in source 编译时,他跟 CMAKE_BINARY_DIR 等变量一致。
PROJECT_SOURCE_DIR 跟其他指令稍有区别,现在,你可以理解为他们是一致的。

### 3. CMAKE_CURRENT_SOURCE_DIR
指的是当前处理的 CMakeLists.txt 所在的路径,比如上面我们提到的 src 子目录。

### 4. CMAKE_CURRRENT_BINARY_DIR
如果是 in-source 编译,它跟 CMAKE_CURRENT_SOURCE_DIR 一致,如果是 out-of-source 编译,他指的是 target 编译目录。
使用我们上面提到的 ADD_SUBDIRECTORY(src bin)可以更改这个变量的值。
使用 SET(EXECUTABLE_OUTPUT_PATH <新路径>)并不会对这个变量造成影响,它仅仅修改了最终目标文件存放的路径。

### 5. CMAKE_CURRENT_LIST_FILE
输出调用这个变量的 CMakeLists.txt 的完整路径

### 6. CMAKE_CURRENT_LIST_LINE
输出这个变量所在的行

### 7. CMAKE_MODULE_PATH
这个变量用来定义自己的 cmake 模块所在的路径。如果你的工程比较复杂,有可能会自己编写一些 cmake 模块,这些 cmake 模块是随你的工程发布的,为了让 cmake 在处理CMakeLists.txt 时找到这些模块,你需要通过 SET 指令,将自己的 cmake 模块路径设置一下。
比如
SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
这时候你就可以通过 INCLUDE 指令来调用自己的模块了。

### 8. EXECUTABLE_OUTPUT_PATH 和 LIBRARY_OUTPUT_PATH
分别用来重新定义最终结果的存放目录,前面我们已经提到了这两个变量。

### 9. PROJECT_NAME
返回通过 PROJECT 指令定义的项目名称。

### 10. CMAKE_BUILD_TYPE
生成 debug 版和 release 版的程序
... ...

### 11. CMAKE_C_COMPILER
指定C编译器，通常，CMake运行时能够自动检测C语言编译器。进行嵌入式系统开发时，通常需要设置此变量，指定交叉编译器。

### 12. CMAKE_CXX_COMPILER
指定C++编译器

### 13. CMAKE_C_FLAGS
指定编译C文件时编译选项,比如-g指定产生调试信息。也可以通过add_definitions命令添加编译选项。

### 14. EXECUTABLE_OUTPUT_PATH
指定可执行文件存放的路径。

### 15. LIBRARY_OUTPUT_PATH
指定库文件放置的路径

# 四、 调用环境变量
- 使用$ENV{NAME}指令就可以调用系统的环境变量了。  
eg: MESSAGE(STATUS “HOME dir: $ENV{HOME}”)
- 设置环境变量的方式是: SET(ENV{变量名} 值)
- 1. CMAKE_INCLUDE_CURRENT_DIR
自动添加 CMAKE_CURRENT_BINARY_DIR 和 CMAKE_CURRENT_SOURCE_DIR 到当前处理的 CMakeLists.txt。相当于在每个 CMakeLists.txt 加入:INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}${CMAKE_CURRENT_SOURCE_DIR})
- 2. CMAKE_INCLUDE_DIRECTORIES_PROJECT_BEFORE
将工程提供的头文件目录始终至于系统头文件目录的前面,当你定义的头文件确实跟系统发生冲突时可以提供一些帮助。
- 3. CMAKE_INCLUDE_PATH 和 CMAKE_LIBRARY_PATH

# 五、 系统信息
- 1. CMAKE_MAJOR_VERSION，CMAKE 主版本号,比如 2.4.6 中的 2
- 2. CMAKE_MINOR_VERSION，CMAKE 次版本号,比如 2.4.6 中的 4
- 3. CMAKE_PATCH_VERSION，CMAKE 补丁等级,比如 2.4.6 中的 6
- 4. CMAKE_SYSTEM，系统名称,比如 Linux-2.6.22
- 5. CMAKE_SYSTEM_NAME，不包含版本的系统名,比如 Linux
- 6. CMAKE_SYSTEM_VERSION，系统版本,比如 2.6.22
- 7. CMAKE_SYSTEM_PROCESSOR，处理器名称,比如 i686.
- 8. UNIX，在所有的类 UNIX 平台为 TRUE,包括 OS X 和 cygwin
- 9. WIN32，在所有的 win32 平台为 TRUE,包括 cygwin

# 六、 主要的开关选项
### 1. CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS
用来控制 IF ELSE 语句的书写方式,在下一节语法部分会讲到。
### 2. BUILD_SHARED_LIBS
这个开关用来控制默认的库编译方式,如果不进行设置,使用 ADD_LIBRARY 并没有指定库类型的情况下,默认编译生成的库都是静态库。
如果 SET(BUILD_SHARED_LIBS ON)后,默认生成的为动态库。
### 3. CMAKE_C_FLAGS
设置 C 编译选项,也可以通过指令 ADD_DEFINITIONS()添加。
### 4. CMAKE_CXX_FLAGS
设置 C++编译选项,也可以通过指令 ADD_DEFINITIONS()添加。

> 本章介绍了一些较常用的 cmake 变量,这些变量仅仅是所有 cmake 变量的很少一部分,目
前 cmake 的英文文档也是比较缺乏的,如果需要了解更多的 cmake 变量,更好的方式是阅
读一些成功项目的 cmake 工程文件,比如 KDE4 的代码。

# 七、 常用指令
## (1)基本指令
### 1. ADD_DEFINITIONS
向 C/C++编译器添加-D 定义,比如:
ADD_DEFINITIONS(-DENABLE_DEBUG -DABC),参数之间用空格分割。
如果你的代码中定义了#ifdef ENABLE_DEBUG #endif,这个代码块就会生效。
如果要添加其他的编译器开关,可以通过 CMAKE_C_FLAGS 变量和 CMAKE_CXX_FLAGS 变量设置。

### 2. ADD_DEPENDENCIES
定义 target 依赖的其他 target,确保在编译本 target 之前,其他的 target 已经被构建。
ADD_DEPENDENCIES(target-name depend-target1 depend-target2 ...)

让一个顶层目标依赖于其他的顶层目标。一个顶层目标是由命令ADD_EXECUTABLE，ADD_LIBRARY，或者ADD_CUSTOM_TARGET产生的目标。为这些命令的输出引入依赖性可以保证某个目标在其他的目标之前被构建。查看ADD_CUSTOM_TARGET和ADD_CUSTOM_COMMAND命令的DEPENDS选项，可以了解如何根据自定义规则引入文件级的依赖性。查看SET_SOURCE_FILES_PROPERTIES命令的OBJECT_DEPENDS选项，可以了解如何为目标文件引入文件级的依赖性。

### 3. ADD_EXECUTABLE、ADD_LIBRARY、ADD_SUBDIRECTORY
- ADD_EXECUTABLE(可执行文件名  生成该可执行文件的源文件)
说明源文件需要编译出的可执行文件名  
例： ADD_EXECUTABLE(hello ${SRC_LIST})  
说明SRC_LIST变量中的源文件需要编译出名为hello的可执行文件
- ADD_LIBRARY(libname [SHARED|STATIC|MODULE] [EXCLUDE_FROM_ALL] source1 source2 ... sourceN)
生成动态静态库  
例： ADD_LIBRARY(hello SHARED ${LIBHELLO_SRC})  
ADD_SUBDIRECTORY(src_dir [binary_dir] [EXCLUDE_FROM_ALL])  
向当前工程添加存放源文件的子目录，并可以指定中间二进制和目标二进制的存放位置EXCLUDE_FROM_ALL含义：将这个目录从编译过程中排除

### 4. ADD_TEST 与 ENABLE_TESTING 指令。
- ENABLE_TESTING 指令用来控制 Makefile 是否构建 test 目标,涉及工程所有目录。语法很简单,没有任何参数,ENABLE_TESTING(),一般情况这个指令放在工程的主CMakeLists.txt 中.
- ADD_TEST 指令的语法是:
ADD_TEST(testname Exename arg1 arg2 ...)  
- testname 是自定义的 test 名称,Exename 可以是构建的目标文件也可以是外部脚本等等。后面连接传递给可执行文件的参数。如果没有在同一个 CMakeLists.txt 中打开ENABLE_TESTING()指令,任何 ADD_TEST 都是无效的。
- 比如我们前面的 Helloworld 例子,可以在工程主 CMakeLists.txt 中添加ADD_TEST(mytest ${PROJECT_BINARY_DIR}/bin/main) ENABLE_TESTING()  
生成 Makefile 后,就可以运行 make test 来执行测试了。

### 5. AUX_SOURCE_DIRECTORY
- 基本语法是:
AUX_SOURCE_DIRECTORY(dir VARIABLE)
- 作用是发现一个目录下所有的源代码文件并将列表存储在一个变量中,这个指令临时被用来自动构建源文件列表。因为目前 cmake 还不能自动发现新添加的源文件。
- 比如  
AUX_SOURCE_DIRECTORY(. SRC_LIST)  
ADD_EXECUTABLE(main ${SRC_LIST})  
- 你也可以通过后面提到的 FOREACH 指令来处理这个 LIST

### 6. CMAKE_MINIMUM_REQUIRED
- 其语法为 CMAKE_MINIMUM_REQUIRED(VERSION versionNumber [FATAL_ERROR])
- 比如 CMAKE_MINIMUM_REQUIRED(VERSION 2.5 FATAL_ERROR)
- 如果 cmake 版本小与 2.5,则出现严重错误,整个过程中止。

### 7. EXEC_PROGRAM
> 在 CMakeLists.txt 处理过程中执行命令,并不会在生成的 Makefile 中执行。

- 具体语法为:
```
EXEC_PROGRAM(Executable [directory in which to run]
                [ARGS <arguments to executable>]
                [OUTPUT_VARIABLE <var>]
                [RETURN_VALUE <var>])
```
- 用于在指定的目录运行某个程序,通过 ARGS 添加参数,如果要获取输出和返回值,可通过OUTPUT_VARIABLE 和 RETURN_VALUE 分别定义两个变量.
- 这个指令可以帮助你在 CMakeLists.txt 处理过程中支持任何命令,比如根据系统情况去修改代码文件等等。
- 举个简单的例子,我们要在 src 目录执行 ls 命令,并把结果和返回值存下来。  
可以直接在 src/CMakeLists.txt 中添加:
```
EXEC_PROGRAM(ls ARGS "*.c" OUTPUT_VARIABLE LS_OUTPUT RETURN_VALUE
LS_RVALUE)
IF(not LS_RVALUE)
MESSAGE(STATUS "ls result: " ${LS_OUTPUT})
ENDIF(not LS_RVALUE)
```
- 在 cmake 生成 Makefile 的过程中,就会执行 ls 命令,如果返回 0,则说明成功执行,那么就输出 ls *.c 的结果。关于 IF 语句,后面的控制指令会提到。

### 8. FILE 指令
文件操作指令,基本语法为:
```
  file(WRITE filename "message to write"... )
  file(APPEND filename "message to write"... )
  file(READ filename variable [LIMIT numBytes] [OFFSET offset] [HEX])
  file(STRINGS filename variable [LIMIT_COUNT num]
       [LIMIT_INPUT numBytes] [LIMIT_OUTPUT numBytes]
       [LENGTH_MINIMUM numBytes] [LENGTH_MAXIMUM numBytes]
       [NEWLINE_CONSUME] [REGEX regex]
       [NO_HEX_CONVERSION])
  file(GLOB variable [RELATIVE path] [globbing expressions]...)
  file(GLOB_RECURSE variable [RELATIVE path]
       [FOLLOW_SYMLINKS] [globbing expressions]...)
  file(RENAME <oldname> <newname>)
  file(REMOVE [file1 ...])
  file(REMOVE_RECURSE [file1 ...])
  file(MAKE_DIRECTORY [directory1 directory2 ...])
  file(RELATIVE_PATH variable directory file)
  file(TO_CMAKE_PATH path result)
  file(TO_NATIVE_PATH path result)
  file(DOWNLOAD url file [TIMEOUT timeout] [STATUS status] [LOG log]
       [EXPECTED_MD5 sum] [SHOW_PROGRESS])
```
- WRITE选项将会写一条消息到名为filename的文件中。如果文件已经存在，该命令会覆盖已有的文件；如果文件不存在，它将创建该文件。
- APPEND选项和WRITE选项一样，将会写一条消息到名为filename的文件中，只是该消息会附加到文件末尾。
- READ选项将会读一个文件中的内容并将其存储在变量里。读文件的位置从offset开始，最多读numBytes个字节。如果指定了HEX参数，二进制代码将会转换为十六进制表达方式，并存储在变量里。
- STRINGS将会从一个文件中将一个ASCII字符串的list解析出来，然后存储在variable变量中。文件中的二进制数据会被忽略。回车换行符会被忽略。它也可以用在Intel的Hex和Motorola的S-记录文件；读取它们时，它们会被自动转换为二进制格式。可以使用NO_HEX_CONVERSION选项禁止这项功能。LIMIT_COUNT选项设定了返回的字符串的最大数量。LIMIT_INPUT设置了从输入文件中读取的最大字节数。LIMIT_OUTPUT设置了在输出变量中存储的最大字节数。LENGTH_MINIMUM设置了要返回的字符串的最小长度；小于该长度的字符串会被忽略。LENGTH_MAXIMUM设置了返回字符串的最大长度；更长的字符串会被分割成不长于最大长度的字符串。NEWLINE_CONSUME选项允许新行被包含到字符串中，而不是终止它们。REGEX选项指定了一个待返回的字符串必须满足的正则表达式。
- GLOB选项将会为所有匹配查询表达式的文件生成一个文件list，并将该list存储进变量variable里。文件名查询表达式与正则表达式类似，只不过更加简单。如果为一个表达式指定了RELATIVE标志，返回的结果将会是相对于给定路径的相对路径。
- GLOB_RECURSE选项将会生成一个类似于通常的GLOB选项的list，只是它会寻访所有那些匹配目录的子路径并同时匹配查询表达式的文件。作为符号链接的子路径只有在给定FOLLOW_SYMLINKS选项或者cmake策略CMP0009被设置为NEW时，才会被寻访到。参见cmake --help-policy CMP0009 查询跟多有用的信息。
- MAKE_DIRECTORY选项将会创建指定的目录，如果它们的父目录不存在时，同样也会创建。（类似于mkdir命令——译注）
- RENAME选项对同一个文件系统下的一个文件或目录重命名。（类似于mv命令——译注）
- REMOVE选项将会删除指定的文件，包括在子路径下的文件。（类似于rm命令——译注）
- REMOVE_RECURSE选项会删除给定的文件以及目录，包括非空目录。（类似于rm -r 命令——译注）
- RELATIVE_PATH选项会确定从direcroty参数到指定文件的相对路径。
- TO_CMAKE_PATH选项会把path转换为一个以unix的 / 开头的cmake风格的路径。输入可以是一个单一的路径，也可以是一个系统路径，比如"$ENV{PATH}"。注意，在调用TO_CMAKE_PATH的ENV周围的双引号只能有一个参数(Note the double quotes around the ENV call TO_CMAKE_PATH only takes one argument. 原文如此。quotes和后面的takes让人后纠结，这句话翻译可能有误。欢迎指正——译注)。
- TO_NATIVE_PATH选项与TO_CMAKE_PATH选项很相似，但是它会把cmake风格的路径转换为本地路径风格：windows下用\，而unix下用/。
- DOWNLOAD 将给定的URL下载到指定的文件中。如果指定了LOG var选项，下载日志将会被输出到var中。如果指定了STATUS var选项，下载操作的状态会被输出到var中。该状态返回值是一个长度为2的list。list的第一个元素是操作的数字返回值，第二个返回值是错误的字符串值。错误信息如果是数字0，操作中没有发生错误。如果指定了TIMEOUT time选项，在time秒之后，操作会超时退出；time应该是整数。如果指定了EXPECTED_MD5 sum选项，下载操作会认证下载的文件的实际MD5和是否与期望值匹配。如果不匹配，操作将返回一个错误。如果指定了SHOW_PROGRESS选项，进度信息会以状态信息的形式被打印出来，直到操作完成。

### 9. INCLUDE 指令
- 用来载入 CMakeLists.txt 文件,也用于载入预定义的 cmake 模块.
```
       INCLUDE(file1 [OPTIONAL])
       INCLUDE(module [OPTIONAL])
```
- OPTIONAL 参数的作用是文件不存在也不会产生错误。
- 你可以指定载入一个文件,如果定义的是一个模块,那么将在 CMAKE_MODULE_PATH 中搜索这个模块并载入。载入的内容将在处理到 INCLUDE 语句时直接执行

## (2) INSTALL 指令
> INSTALL 系列指令已经在前面的章节有非常详细的说明,这里不在赘述,可参考前面的安装部分

## (3) FIND_系列指令
- FIND_系列指令主要包含一下指令:  
FIND_FILE(<VAR> name1 path1 path2 ...)  
VAR 变量代表找到的文件全路径,包含文件名

### 1. FIND_PATH(<VAR> name1 path1 path2 ...)
- 指明头文件查找的路径，原型如下：
- 该命令在参数 path* 指示的目录中查找文件 name1 并将查找到的路径保存在变量 VAR 中

### 2. FIND_LIBRARY(<VAR> name1 path1 path2 ...)
用法同FIND_PATH，表明库查找路径。

### 3. FIND_PROGRAM(<VAR> name1 path1 path2 ...)
VAR 变量代表包含这个程序的全路径。

### 4. FIND_PACKAGE( <name> [major.minor] [QUIET] [NO_MODULE] [ [ REQUIRED | COMPONENTS ] [ componets... ] ] )
- 用来调用预定义在 CMAKE_MODULE_PATH 下的 Find<name>.cmake 模块,你也可以自己定义 Find<name>模块,通过 SET(CMAKE_MODULE_PATH dir)将其放入工程的某个目录中供工程使用。
- 这条命令执行后，CMake 会到变量 CMAKE_MODULE_PATH 指示的目录中查找文件 Findname.cmake 并执行。

### QUIET 参数:  
- 对应于Find<name>.cmake模块中的 NAME_FIND_QUIETLY，如果不指定这个参数，就会执行:  
MESSAGE(STATUS "Found NAME: ${NAME_LIBRARY}")  
指定了QUIET就不输出指令。  

### REQUIRED 参数  
其含义是指是否是工程必须的，如果使用了这个参数,说明是必须的，如果找不到，则工程不能编译。对应于Find<name>.cmake模块中的 NAME_FIND_REQUIRED 变量。

## (4) 控制指令

### 1. IF
```
       IF(expression_r_r)
         # THEN section.
         COMMAND1(ARGS ...)
         COMMAND2(ARGS ...)
         ...
       ELSE(expression_r_r)
         # ELSE section.
         COMMAND1(ARGS ...)
         COMMAND2(ARGS ...)
         ...
       ENDIF(expression_r_r)

```
- 另外一个指令是 ELSEIF,总体把握一个原则,凡是出现 IF 的地方一定要有对应的ENDIF.出现 ELSEIF 的地方,ENDIF 是可选的。

表达式的使用方法如下:
- IF(var),如果变量不是:空,0,N, NO, OFF, FALSE, NOTFOUND 或<var>_NOTFOUND 时,表达式为真。
- IF(NOT var ),与上述条件相反。
- IF(var1 AND var2),当两个变量都为真是为真。
- IF(var1 OR var2),当两个变量其中一个为真时为真。
- IF(COMMAND cmd),当给定的 cmd 确实是命令并可以调用是为真。
- IF(EXISTS dir)或者 IF(EXISTS file),当目录名或者文件名存在时为真。
- IF(file1 IS_NEWER_THAN file2),当 file1 比 file2 新,或者 file1/file2 其中有一个不存在时为真,文件名请使用完整路径。
- IF(IS_DIRECTORY dirname),当 dirname 是目录时,为真。
- IF(variable MATCHES regex)
- IF(string MATCHES regex)  
当给定的变量或者字符串能够匹配正则表达式 regex 时为真。比如:
```
IF("hello" MATCHES "ell")
MESSAGE("true")
ENDIF("hello" MATCHES "ell")
IF(variable LESS number)
IF(string LESS number)
IF(variable GREATER number)
IF(string GREATER number)
IF(variable EQUAL number)
IF(string EQUAL number)
//数字比较表达式
IF(variable STRLESS string)
IF(string STRLESS string)
IF(variable STRGREATER string)
IF(string STRGREATER string)
IF(variable STREQUAL string)
IF(string STREQUAL string)
```
- 按照字母序的排列进行比较.  
IF(DEFINED variable),如果变量被定义,为真。
- 一个小例子,用来判断平台差异:
```
IF(WIN32)
    MESSAGE(STATUS “This is windows.”)
    #作一些 Windows 相关的操作
ELSE(WIN32)
    MESSAGE(STATUS “This is not windows”)
    #作一些非 Windows 相关的操作
ENDIF(WIN32)
```
上述代码用来控制在不同的平台进行不同的控制,但是,阅读起来却并不是那么舒服,ELSE(WIN32)之类的语句很容易引起歧义。  
这就用到了我们在“常用变量”一节提到的 CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS 开关。  
可以 SET(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)  
这时候就可以写成:
```
IF(WIN32)
ELSE()
ENDIF()
```
如果配合 ELSEIF 使用,可能的写法是这样:
```
IF(WIN32)
#do something related to WIN32
ELSEIF(UNIX)
#do something related to UNIX
ELSEIF(APPLE)
#do something related to APPLE
ENDIF(WIN32)
```

### 2. WHILE
WHILE 2. 指令的语法是:
```
        WHILE(condition)
          COMMAND1(ARGS ...)
          COMMAND2(ARGS ...)
          ...
        ENDWHILE(condition)
```
其真假判断条件可以参考 IF 指令。

### 3. FOREACH
FOREACH 指令的使用方法有三种形式:

##### i. 列表
```
FOREACH(loop_var arg1 arg2 ...)
          COMMAND1(ARGS ...)
          COMMAND2(ARGS ...)
          ...
ENDFOREACH(loop_var)
```
像我们前面使用的 AUX_SOURCE_DIRECTORY 的例子
```
AUX_SOURCE_DIRECTORY(. SRC_LIST)
FOREACH(F ${SRC_LIST})
    MESSAGE(${F})
ENDFOREACH(F)
```

#### ii. 范围
```
FOREACH(loop_var RANGE total)
ENDFOREACH(loop_var)
```
从 0 到 total 以1为步进  
举例如下:
```
FOREACH(VAR RANGE 10)
MESSAGE(${VAR})
ENDFOREACH(VAR)
```

#### iii. 范围和步进
```
FOREACH(loop_var RANGE start stop [step])
ENDFOREACH(loop_var)
```
从 start 开始到 stop 结束,以 step 为步进,
举例如下
```
FOREACH(A RANGE 5 15 3)
MESSAGE(${A})
ENDFOREACH(A)
```
