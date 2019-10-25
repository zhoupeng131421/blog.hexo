---
title: SHELL中 $*... 等用法
date: 2019-10-25
tags: [shell, &>/dev/null]
categories: Learning
---

# SHELL 中 `$* $0 $? $! $$ $* $# $@` 等用法

- `$$`: Shell 本身的 PID.
- `$!`: Shell 最后运行的后台process的PID
- `$?`: 最后运行的命令的结束代码，即上一命令的返回值(0 表示没有返回值)
- `$-`: Shell使用的选项，与set功能相同
- `$*`: 所有参数列表，如`"$*"`用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数，此选项参数可超过9个。
- `$@`: 所有参数列表。如`"$@"`用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。
- `$@` 跟 `$*` 类似，但是可以当做数组用
- `$#`: 添加到Shell的参数个数
- `$0`: Shell本身的文件名
- `$1～$n`: 添加到Shell的各参数值。$1是第1参数、$2是第2参数…。

### 举例
- vim variable.sh:
  ```shell
  echo "number:$#"
  echo "scname:$0"
  echo "first :$1"
  echo "second:$2"
  echo "argume:$@"
  ```
- 执行 `./variable.sh aa bb`:
  ```shell
  number:2
  scname:./variable
  first: aa
  second:bb
  argume:aa bb
  ```

- $# 是传给脚本的参数个数
- $0 是脚本本身的名字
- $1是传递给该shell脚本的第一个参数
- $2是传递给该shell脚本的第二个参数
- $@ 是传给脚本的所有参数的列表


# **&>/dev/null**
> 在Linux/Unix中，一般在屏幕上面看到的信息是从stdout (standard output) 或者 stderr (standard error output) 来的。通常在 server 的工作环境下，几乎所有的程序都是 run 在 background 的，所以，为了方便 debug，一般在设计程序时，就把stdout 送到/存到一个档案，把错误的信息 stderr 存到不同的档案。

- 用 /dev/null 2>&1 这样的写法.这条命令的意思是将标准输出和错误输出全部重定向到/dev/null中,也就是将产生的所有信息丢弃.

### > /dev/null
- 这是把 stdout 送到 /dev/null 里面
- 那什麼是 /dev/null 呢，/dev/null 是 Unix/Linux 里的【无底洞】
- 任何的 output 送去了【无底洞】就再也没了。
- 要是你不想看到 output 或者output 太多太大了，有可能把硬碟给挤爆了的时候，程序的设计就会考虑把 output 送到 /dev/null。

### `command > file 2>file` 与 `command > file 2>&1`不同点
- `command > file 2>file` 的意思是将命令所产生的标准输出信息,和错误的输出信息送到file中。
- command  > file 2>file 这样的写法,stdout和stderr都直接送到file中, file会被打开两次,这样stdout和stderr会互相覆盖,这样写相当使用了FD1和FD2两个同时去抢占file 的管道.
- `command >file 2>&1` 这条命令就将stdout直接送向file, stderr 继承了FD1管道后,再被送往file,此时,file 只被打开了一次,也只使用了一个管道FD1,它包括了stdout和stderr的内容.
- 从IO效率上,前一条命令的效率要比后面一条的命令效率要低,所以在编写shell脚本的时候,较多的时候我们会用command > file 2>&1 这样的写法.

### `>/dev/null 2>&1` 详解
> 命令的结果可以通过%>的形式来定义输出

1. `>`代表重定向到哪里，例如：echo "123" > /home/123.txt
2. /dev/null 代表空设备文件
3. 2> 表示stderr标准错误
4. & 表示等同于的意思，2>&1，表示2的输出重定向等同于1
5. 1 表示stdout标准输出，系统默认值是1，所以">/dev/null"等同于 "1>/dev/null"

- \>/dev/null 2>&1也可以写成"1> /dev/null 2> &1"

语句执行过程为：
1. 1>/dev/null ：首先表示标准输出重定向到空设备文件，也就是不输出任何信息到终端，说白了就是不显示任何信息。
2. 2>&1 ：接着，标准错误输出重定向 到标准输出，因为之前标准输出已经重定向到了空设备文件，所以标准错误输出也重定向到空设备文件。
