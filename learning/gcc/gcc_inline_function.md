---
title: gcc_inline_function
date: 2019-11-15
tags: [gcc]
categories: Learning
---

在程序中，通过把一个函数声明为内联（inline）函数，就可以让gcc把函数的代码集成到调用该函数的代码中去。这样处理可以去掉函数调用时进入/退出时间开销，从而肯定能够加快执行速度。因此把一个函数声明为内联函数的主要目的就是能够尽量快速地执行函数体。另外，如果内联函数中有常数值，那么在编译期间gcc就可能用它来进行一些简化操作，因此并非所有内联函数的代码都会被嵌入进去。内联函数方法对程序代码的长度影响并不明显。使用内联函数的程序编译产生的目标代码可能会长一些也可能会短一些，这需要根据具体情况来定。

内联函数嵌入调用者代码中的操作是一种优化操作，因此只有进行优化编译时才会执行代码嵌入处理。若编译过程中没有使用优化选项'-O'，那么内联函数的代码就不会被真正地嵌入到调用者代码中，而是只作为普通函数调用来处理。把一个函数声明为内联函数的方法是在函数声明中使用关键词'inline'，例如内核文件fs/inode.c中的如下函数：
``` c
inline int inc(int *a)
{
  (*a)++;
}
```
函数中的某些语句用法可能会使得内联函数的替换操作无法正常进行，或者不适合进行替换操作。例如使用了可变参数、内存分配函数malloca()、可变长度数据类型变量、非局部goto语句以及递归函数。编译时可以使用选项 -Winline 让gcc对标志成 inline但不能被替换的函数给出警告信息以及不能替换的原因。

当在一个函数定义中既使用inline关键词，又使用static关键词，即像下面文件fs/inode.c中的内联函数定义一样，那么如果所有对该内联函数的调用都被替换而集成在调用者代码中，并且程序中没有引用过该内联函数的地址，则该内联函数自身的汇编代码就不会被引用。在这种情况下，除非我们在编译过程中使用选项 -fkeep-inline-functions，否则gcc就不会再为该内联函数自身生成实际汇编代码。由于某些原因，一些对内联函数的调用并不能被集成到函数中去。特别是在内联函数定义之前的调用语句是不会被替换集成的，并且也都不能是递归定义的函数。如果存在一个不能被替换集成的调用，那么内联函数就会像平常一样被编译成汇编代码。当然，如果程序中有引用内联函数地址的语句，那么内联函数也会像平常一样被编译成汇编代码。因为对内联函数地址的引用是不能被替换的。
``` c
static inline void wait_on_inode(struct m_inode * inode)
{
  cli();
  while (inode->i_lock)
    sleep_on(&inode->i_wait);
  sti();
}
```
请注意，内联函数功能已经被包括在ISO标准C99中，但是该标准定义的内联函数与gcc的定义有较大区别。ISO标准C99的内联函数语义定义等同于这里使用组合关键词 inline 和 static的定义，即'省略'了关键词 static。若在程序中需要使用C99标准的语义，那么就需要使用编译选项-std=gnu99。不过为了兼容起见，在这种情况下还是最好使用 inline 和 static 组合。以后gcc将最终默认使用C99的定义，在希望仍然使用这里定义的语义时，就需要使用选项 -std=gnu89来指定。

若一个内联函数的定义没有使用关键词 static，那么gcc就会假设其他程序文件中也对这个函数有调用。因为一个全局符号只能被定义一次，所以该函数就不能在其他源文件中再进行定义。因此这里对内联函数的调用就不能被替换集成。因此，一个非静态的内联函数总是会被编译出自己的汇编代码来。在这方面，ISO标准C99对不使用 static 关键词的内联函数定义等同于这里使用 static 关键词的定义。

如果在定义一个函数时还指定了 inline 和 extern 关键词，那么该函数定义仅用于内联集成，并且在任何情况下都不会单独产生该函数自身的汇编代码，即使明确引用了该函数的地址也不会产生。这样的一个地址会变成一个外部引用，就好像你仅仅声明了函数而没有定义函数一样。

关键词inline和extern组合在一起的作用几乎类同一个宏定义。使用这种组合方式就是把带有组合关键词的一个函数定义放在.h头文件中，并且把不含关键词的另一个相同函数定义放在一个库文件中。此时头文件中的定义会让绝大多数对该函数的调用被替换嵌入。如果还有未被替换的对该函数的调用，那么就会使用（引用）程序文件中或库中的副本。Linux 0.1x内核源代码中文件include/string.h、lib/strings.c就是这种使用方式的一个例子。例如，string.h中定义了如下函数：  
```c
// 将字符串(src)复制到另一字符串(dest)，直到遇到NULL字符后停止。
// 参数：dest - 目的字符串指针，src - 源字符串指针。%0 - esi(src)，%1 - edi(dest)。
extern inline char * strcpy(char * dest,const char *src)
{
   __asm__('cld\n'   // 清方向位。
       '1:\tlodsb\n\t' // 加载DS:[esi]处1字节→al，并更新esi。
       'stosb\n\t'  // 存储字节al→ES:[edi]，并更新edi。
       'testb %%al,%%al\n\t' // 刚存储的字节是0？
       'jne 1b'  // 不是则向后跳转到标号1处，否则结束。
       ::'S' (src),'D' (dest):'si','di','ax');
   return dest;   // 返回目的字符串指针。
}
```
而在内核函数库目录中，lib/strings.c文件把关键词 inline 和 extern 都定义为空，如下所示。因此实际上就在内核函数库中又包含了string.h文件所有这类函数的一个副本，即又对这些函数重新定义了一次，并且'消除'了两个关键词的作用。
```c
#define extern  // 定义为空。
#define inline // 定义为空。
#define __LIBRARY__
#include <string.h>
```
此时库函数中重新定义的上述strcpy()函数变成如下形式：
```c
char * strcpy(char * dest,const char *src)  // 去掉了关键词 inline 和 extern。
{
   __asm__('cld\n'   // 清方向位。
       '1:\tlodsb\n\t'  // 加载DS:[esi]处1字节→al，并更新esi。
       'stosb\n\t'  // 存储字节al→ES:[edi]，并更新edi。
       'testb %%al,%%al\n\t'  // 刚存储的字节是0？
       'jne 1b'   // 不是则向后跳转到标号1处，否则结束。
       ::'S' (src),'D' (dest):'si','di','ax');
    return dest;   // 返回目的字符串指针。
}
```
来源： http://my.oschina.net/senmole/blog/52002