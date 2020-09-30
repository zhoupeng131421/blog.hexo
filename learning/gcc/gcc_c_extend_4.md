---
title: gcc_c_extend_4
date: 2019-11-14
tags: [gcc]
categories: Learning
---

## 返回值的复合语句
- 复合语句是大括号包围的一个语句块，在复合语句内你可以声明自己的变量，如下例所示：
```c
{
    int a = 5;
    int b;
    b = a + 5;
}
```
- 在GNU C中，圆括号包围的复合语句可以生成返回值，返回的结果的类型和值是复合语句的最后一句的类型和值。如下例所示，它会返回值8：
```c
rslt = ({
            int a = 5;
            int b;
            b = a + 3;
       });
```
- 下面的宏返回一个等于或大于x的偶数
```c
#define even(x)    (2 * ((x) / 2) == (x) ? (x) : (x) + 1)
```
- 在使用even(i++)这样的表达式时应付产生副用，下面的宏实现同样的功能，使用了GCC的扩展，在复合语句里声明了一个临时变量：
```c
#define evenint(x)    \
      ({int y = x; \
        (2 * (y / 2) == y ? y : y + 1); \
     })
```
注意：这种功能扩展在C++中会产生问题。问题来自于宏内部的临时变量的析构函数，临时变量会在赋值前被析构。

## 条件操作数的省略
- 如下例中的条件表达式：
```c
x = y ? y : z;
```
- 在GCC扩展中，可以写成下面的形式：
```c
x = y ? : z;
```

## 构造函数调用
- GCC提供了三个内置函数用于将当前函数的参数直接传递给另一个函数，并将另一个函数的执行结果返回给调用者。
- 下面的函数找到并返回参数的信息：
```c
void *__builtin_apply_args(void);
```
- 一旦获得了参数的信息，可以用下面的函数去构造调用所需要的栈（stack）并调用该函数。
```c
void *__builtin_apply(void (*func)(), void *arguments, int size);
```
- 第一个参数是要调用函数的地址，第二个是由__builtin_apply_args()返回调用参数的信息，用于构造调用的参数。size参数为从当前堆栈框架（stack frame）复制到新堆栈框架中的字节数，该值应该足够大，以容纳调用传送的参数和返回值。
- 一旦调用完成，可以用下面的函数将func的执行结果返回给调用者。
```c
__builtin_return(void *result);
```
result为__builtin_apply返回的值。
- 下面的程序调用了passthrough()，它通过内建的三个函数来调用average()函数，并将其结果返回给它的调用者。
```c
/* args.c */
#include <stdio.h>
int passthrough();
int average();
int main(int argc, char *argv[])
{
    int result;
    result = passthrough(1, 7, 10);
    printf("result = %d\n", result);
    return 0;
}
int passthrough(int a, int b, int c)
{
    void *record;
    void *playback;
    void (* fn)() = (void (*)()) average;

    record = __builtin_apply_args();
    playback = __builtin_apply(fn, record, 128);
    __builtin_return(playback);
}
int average(int a, int b, int c)
{
    return ((a + b + c) / 3);
}
```
- 如果用省略号创建变长参数列表，返回值指向void指针，函数passthrough()只需要知道要函数的地址，就可以调用任何函数而返回任何类型。
- 来源： http://my.oschina.net/senmole/blog/51997
