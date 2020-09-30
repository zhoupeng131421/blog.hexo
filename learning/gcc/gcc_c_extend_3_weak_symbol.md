---
title: gcc_c_extend_3_weak_symbol
date: 2019-11-13
tags: [gcc]
categories: Learning
---

## 关于弱符号的解释：
- 若两个或两个以上全局符号（函数或变量名）名字一样，而其中之一声明为weak symbol（弱符号），则这些全局符号不会引发重定义错误。链接器会忽略弱符号，去使用普通的全局符号来解析所有对这些符号的引用，但当普通的全局符号不可用时，链接器会使用弱符号。当有函数或变量名可能被用户覆盖时，该函数或变量名可以声明为一个弱符号。当weak和alias属性连用时，还可以声明弱别名。

## 弱别名的使用例子：
```c
//weak.c
#include <stdio.h>
void symbol1()
{
    printf("%s\n",__FUNCTION__);
}
//symbol222.c
void symbol222()
{
    printf("%s\n",__FUNCTION__);
}
//void symbol1() __attribute__ ((weak,alias("symbol222")));  //这一包与下面的asm()一句是等效的。
int main()
{
    asm(".weak symbol1\n\t .set symbol1, symbol222\n\t");
    symbol1();
    return 0;
}
```
- 用下面的命令编译运行会输出symbol1
```c
$ gcc -o weak weak.c symobl222.c
$ ./weak
output：symbol1
```
- 当不链接weak.c，即在symbol1函数为定义时，应用用symbol1的弱别名symbol222代替symbol1。
- 用下面的命令编译运行会输出symbol222：
```c
$ gcc -o weak symbol222.c
$ ./weak
output: symbol222
```

## 弱符号的例子：
```c
//weak2.c
#include <stdio.h>
extern void symbol1() __attribute__((weak));
void symbol1()
{
    printf("%s.%s\n",__FILE__,__FUNCTION__);
}
int main()
{
    //asm(".weak symbol1\n\t .set symbol1, symbol222\n\t");
    symbol1();
    return 0;
}
//strong.c
#include <stdio.h>
void symbol1()
{
    printf("%s.%s\n",__FILE__,__FUNCTION__);
}
```
- 编译运行：
- 当不编译链接strong.c时：
```c
$ gcc -o weakstrong weak2.c
$ ./weakstrong
output: weak2.c symbol1
```
- 当链接strong.c时，会用strong.c中的强符号symbol1代替weak2.c的的弱符号symbol1：
```c
$ gcc -o weakstrong weak2.c strong.c
$ ./weakstrong
output: strong.c symbol1
```
- 当有两个函数同名时，则使用强符号（也叫全局符号）来代替弱符号。

来源： http://my.oschina.net/senmole/blog/50887
