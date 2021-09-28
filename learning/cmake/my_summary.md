---
title: my camke summary
date: 2021-5-8
tags: [cmake]
categories: 工具
---

# Function
## cmake_minimum_required (VERSION xxx)
- 指定cmake最小版本

## project()
- Set a name, version, and enable languages for the entire project.
- `project(<PROJECT-NAME> [LANGUAGES] [<language-name>...])`
- `project(<PROJECT-NAME> [VERSION <major>[.<minor>[.<patch>[.<tweak>]]]] [LANGUAGES <language-name>...])`

## add_executable()
- `add_executable(<name> [WIN32] [MACOSX_BUNDLE] [EXCLUDE_FROM_ALL] [source1] [source2 ...])`
- Adds an executable target called `<name>` to be built from the source files listed in the command invocation

## target_link_libraries()
- `target_link_libraries(<target> ... <item>... ...)`
- `target_link_libraries(<target> <PRIVATE|PUBLIC|INTERFACE> <item>... [<PRIVATE|PUBLIC|INTERFACE> <item>...]...)`
- Specify libraries or flags to use when linking a given target and/or its dependents. Usage requirements from linked library targets will be propagated. Usage requirements of a target's dependencies affect compilation of its own sources.
- The `PUBLIC`, `PRIVATE` and INTERFACE keywords can be used to specify both the link dependencies and the link interface in one command. Libraries and targets following `PUBLIC` are linked to, and are made part of the link interface. Libraries and targets following `PRIVATE` are linked to, but are not made part of the link interface. Libraries following INTERFACE are appended to the link interface and are not used for linking `<target>`
    - reference: https://zhuanlan.zhihu.com/p/82244559

## set_target_properties()
- `set_target_properties(target1 target2 ...  PROPERTIES prop1 value1  prop2 value2 ...)`
- Sets properties on targets. The syntax for the command is to list all the targets you want to change, and then provide the values you want to set next. You can use any prop value pair you want and extract it later with the get_property() or get_target_property() command.

## target_link_options()
- Add options to the link step for an executable, shared library or module library target.
```shell
target_link_options(<target> [BEFORE]
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```
- The named `target` must have been created by a command such as add_executable() or add_library() and must not be an ALIAS target.

## add_library()
- Add a library to the project using the specified source files.
- `Nromal` libraries:
    - A `STATIC` library may be marked with the FRAMEWORK target property to create a static Framework.
    ```shell
    add_library(<name> [STATIC | SHARED | MODULE]
                [EXCLUDE_FROM_ALL]
                [<source>...])
    ```
- `object` libraries:
    - `add_library(<name> OBJECT [<source>...])`
    - Creates an Object Library. An object library compiles source files but does not archive or link their object files into a library.
- `interface`, `imported`, and `alias` libraries...

## target_include_directories()
- Add include directories to a target.
```shell
target_include_directories(<target> [SYSTEM] [AFTER|BEFORE]
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```
- `PRIVATE` and `PUBLIC` items will populate the `INCLUDE_DIRECTORIES` property of `target`. `PUBLIC` and `INTERFACE` items will populate the `INTERFACE_INCLUDE_DIRECTORIES` property of `target`.

## add_custom_command()
- Add a custom build rule to the generated build system.
```shell
add_custom_command(OUTPUT output1 [output2 ...]
                   COMMAND command1 [ARGS] [args1...]
                   [COMMAND command2 [ARGS] [args2...] ...]
                   [MAIN_DEPENDENCY depend]
                   [DEPENDS [depends...]]
                   [BYPRODUCTS [files...]]
                   [IMPLICIT_DEPENDS <lang1> depend1
                                    [<lang2> depend2] ...]
                   [WORKING_DIRECTORY dir]
                   [COMMENT comment]
                   [DEPFILE depfile]
                   [JOB_POOL job_pool]
                   [VERBATIM] [APPEND] [USES_TERMINAL]
                   [COMMAND_EXPAND_LISTS])

```
- `DEPENDS` Specify files on which the command depends. 

## add_custom_target()
- Add a target with no output so it will always be built.
```shell
add_custom_target(Name [ALL] [command1 [args1...]]
                  [COMMAND command2 [args2...] ...]
                  [DEPENDS depend depend depend ... ]
                  [BYPRODUCTS [files...]]
                  [WORKING_DIRECTORY dir]
                  [COMMENT comment]
                  [JOB_POOL job_pool]
                  [VERBATIM] [USES_TERMINAL]
                  [COMMAND_EXPAND_LISTS]
                  [SOURCES src1 [src2...]])
```
- `ALL` Indicate that this target should be added to the default build target so that it will be run every time

## file()
- File manipulation command.
- This command is dedicated to file and path manipulation requiring access to the filesystem.
- For other path manipulation, handling only syntactic aspects, have a look at cmake_path() command.
```shell
Reading
  file(READ <filename> <out-var> [...])
  file(STRINGS <filename> <out-var> [...])
  file(<HASH> <filename> <out-var>)
  file(TIMESTAMP <filename> <out-var> [...])
  file(GET_RUNTIME_DEPENDENCIES [...])

Writing
  file({WRITE | APPEND} <filename> <content>...)
  file({TOUCH | TOUCH_NOCREATE} [<file>...])
  file(GENERATE OUTPUT <output-file> [...])
  file(CONFIGURE OUTPUT <output-file> CONTENT <content> [...])

Filesystem
  file({GLOB | GLOB_RECURSE} <out-var> [...] [<globbing-expr>...])
  file(RENAME <oldname> <newname>)
  ...

...
```
- `FileSystem`: Generate a list of files that match the `globbing-expressions` and store it into the `variable`.
    -  If the `CONFIGURE_DEPENDS` flag is specified, CMake will add logic to the main build system check target to rerun the flagged GLOB commands at build time. If any of the outputs change, CMake will regenerate the build system.


## set()
- `set(<variable> <value> [[CACHE <type> <docstring> [FORCE]] | PARENT_SCOPE])`
- Within CMake sets `variable` to the value `value`

## string(REPLACE xxx)
- `string(REPLACE <match_string> <replace_string> <output variable> <input> [<input>...])`
    - 字符串替换：`input` 中的 `match_string` 替换为 `replace_string` 结果输出的 `output variable`

## option()
- `option(<option_variable> "help string describing option" [initial value])`
- Provides an option that the user can optionally select.
- Provide an option for the user to select as ON or OFF. If no initial value is provided, OFF is used.

## add_subdirectory()
- add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
- Adds a subdirectory to the build.
- The `source_dir` specifies the directory in which the source CMakeLists.txt and code files are located.
- The `binary_dir` specifies the directory in which to place the output files.

## enable_language()
- `enable_language(<lang> [OPTIONAL] )`
- This command enables support for the named language in CMake. This is the same as the project command but does not create any of the extra variables that are created by the project command. Example languages are CXX, C, Fortran.

## enable_testing()
- Enables testing for this directory and below.
- This command should be in the source directory root because ctest expects to find a test file in the build directory root.
- This command is automatically invoked when the CTest module is included, except if the BUILD_TESTING option is turned off.

# Variable

## CMAKE_xx_COMPILER_ID
- Compiler identification string. A short string unique to the compiler vendor. 

## CMAKE_CURRENT_SOURCE_DIR
- The path to the source directory currently being processed.

## CMAKE_CURRENT_BINARY_DIR
- The path to the binary directory currently being processed.