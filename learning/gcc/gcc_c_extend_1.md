---
title: gcc_c_extend_1
date: 2019-11-11
tags: [gcc]
categories: Learning
---

## 一、对齐信息 `__alignof__`

可以使用 __alignof__ 关键字查询有关类型或变量的对齐信息。
语法
```c
__alignof__(type)
__alignof__(expr)
```
其中： `type`是类型; `expr`是左值。
返回值: `__alignof__(type)` 返回类型 type 的对齐要求；如果没有对齐要求，则返回 1。
       `__alignof__(expr)` 返回左值 expr 的类型的对齐要求；如果没有对齐要求，则返回 1。
示例:
```c
int Alignment_0(void)
{
    return __alignof__(int);
}
```
## 二、匿名联合
在结构中可以声明某个联合而不用指出名字，这样可以直接使用联合成员就好像它们是结构中的成员一样。下面的例子为相同的四个字节提供两个名字和两个数据类型：
```c
struct {
    char code;
    union {
        char chid[4];
        int numid;
    };
    char *name;
} morx;
```
结构成员可以使用morx.code、morx.chid、morx.numid和morx.name访问。

## 三、变长数组
数组的大小可以声明为动态值，在运行是才指定尺寸。使用表达式数组下标可以实现变成数组。
下面的函数可将输入的两个字符串串联起来，并在其中插入一个空格。
```c
void combine(char *str1, char *str2)
{
    char outstr[strlen(str1) + strlen(str2) + 2];

    strcpy(outstr, str1);
    strcat(outstr, " ");
    strcat(outstr, str2);
    printf("%s\n", outstr);
}
```
变长数组也可作为参数传递，如下例所示：
```c
void fillarray(int length, char letters[length])
{
    int i;
    char character = 'A';

    for (i = 0; i < length; i++)
        letters[i] = character++;
}
```
通过向前声明上例中的参数顺序还可以颠倒过来,这样在读数组letters的时候就已经知道length的类型了，如下：
```c
void fillarray(int length; char letters[length], int length);
```
向前声明的数目可以为任意数目（由逗号或分号分隔），只要最后一个跟着的是分号就可以了。

## 四、零长数组：
GNU C允许声明长度为零的数组，用于创建变成结构。只有当零长度的数组是结构struct的最后一个成员是才有意义。下面的程序展示了零长数组的用法：
```c
/* zarray.c */
#include <stdio.h>
typedef struct {
    int size;
    char string[0];
} vlen;

int main(int argc, char *argv[])
{
    int i;
    int count = 22;
    char letter = 'a';

    vlen *line = (vlen *) malloc(sizeof(vlen) + count);
    line->size = count;
    for (i = 0; i < count; i++)
        line->string[i] = letter++;

    printf("sizeof(vlen) = %d\n", sizeof(vlen));

    for (i = 0; i < line->size; i++)
        printf("%c ", line->string[i]);
    printf("\n");

    return (0);
}
```
将数组定义为未完成类型也能实现同样功能，而且这种方法还能满足标准C，不仅如此，对于前面的例子而言，它的使用方法也完全一致，而且还有一点好处，数组的大小可以在初始化的时候指定，如下例所示，这是数组大小被设为4个字符：
```c
#include <stdio.h>

typedef struct {
    int size;
    char string[];
} vlen;

vlen initvlen = { 4, {'a', 'b', 'c', 'd'}};

int main()
{
    int i;

    printf("sizeof(vlen) = %d\n", sizeof(vlen));
    printf("sizeof(initvlen) = %d\n", sizeof(initvlen));
    for (i = 0; i < initvlen.size; i++)
        printf("%c ", initvlen.string[i]);
    printf("\n");
    return(0);
}
```
上面的输出是：
```c
sizeof(vlen)=4
sizeof(initvlen)=4
a b c d
```
来源： http://my.oschina.net/senmole/blog/50617
