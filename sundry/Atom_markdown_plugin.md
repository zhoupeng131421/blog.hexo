---
title: Atom markdown 插件
date: 2019-10-15
tags: [atom, markdwon]
categories: 随笔
---

# 1、Atom自带markdown-preview
> 功能太少，需要大量拓展。

# 2、markdown-preview-plus
> 功能还不错，但是其中的滚动条插件markdown-scroll-sync和最新版atom不兼容了，1年多未更新，github上全是那个bug的issue。

# 3、markdown-preview-enhanced
> 推荐使用这个，自带mermaid绘图和toc目录等功能，较为全面。

# 4、墙内安装markdown-preview-enhanced
- ATOM自带的settings-install-markdown-preview-enhanced的方法肯定是最简单可行的，但是由于某墙的存在基本下不下来包。于是使用手动方式安装插件：
``` shell
　　a) 下载并安装node.js（为了使用npm命令）
　　b) cd ~/.atom/packages
　　c) git clone https://github.com/shd101wyy/markdown-preview-enhanced.git
　　d) cd markdown-preview-enhanced
　　e) npm install
　　#如果不是在.atom/packages下执行的加一句 apm link，会创建一个连接到.atom/packages目录下。
```
# 5、开始使用
- a) mermaid使用简介：`https://blog.csdn.net/wangyaninglm/article/details/52887045`

- b) toc使用方式：在随便一个地方加上[toc]就行
- c) markdown使用简介：https://shd101wyy.github.io/markdown-preview-enhanced/#/zh-cn/markdown-basics　　介绍比较详细，给scala代码块都能加亮这玩意我还是第一次知道。后来实验了一下，竟然能给Dockerfile加亮，nmdwsm。使用方式如下：
**\```Dockerfile
```**
