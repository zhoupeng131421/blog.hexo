---
title: Window 下 certutil 工具查看文件 MD5 值
date: 2019-10-15
tags: [windows, certutil, MD5]
categories: 随笔
---

# window下查看文件MD5值
* 有时候因为某些原因我们需要查看文件的MD5值，在Linux下这个就非常简单，只需要用md5sum命令即可，但是在Windows上却不知道对应的命令。
* window 也有对应命令，而且该命令还可以查看SHA1值和SHA256值的功能。
命令如下：
```shell
certutil -hashfile filename MD5
certutil -hashfile filename SHA1
certutil -hashfile filename SHA256
```
