---
title: git 详解
date: 2019-10-18
tags: [git]
categories: 工具
---

# UPDATE GIT
>yum源git版本比较老，所以用源码方式安装

1. 安装依赖包
``` shell
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
yum install  gcc perl-ExtUtils-MakeMaker
```
2. uninstall old version
`yum remove git`
3. Download and unzip
```shell
cd /usr/local/src
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.23.0.tar.gz
tar -zxvf git-2.23.0.tar.gz
```
4. compile and install
```shell
cd git-2.23.0
make prefix=/usr/local/git all
make prefix=/usr/local/git install

vim /etc/profile:
  PATH=$PATH:/usr/local/git/bin
  export PATH
```
refresh PATH:`source /etc/profile`


# 常用配置
```shell
git config --global user.name xxx
git config --global user.email xxx
git config --global core.quotepath false    #中文文件名乱码
git config --global credential.helper manager #保存项目相关github 的用户名、密码 信息
git config --global color.ui true
```

# 常用命令
```shell
git clone xxx
git clone --recursive xxx # clone all submodules

git branch -av
git checkout branch -b local_branch

git add xxx
git commit -s   #commit中添加签名
git commit -m "xxx"   #直接添加commit message
git commit --amend  #追加提交，不增加commit id的情况下，将新增加代码添加到上一次commit id中

git push dest src

merge:
  git checkout master
  git merge develop   # merge develop to master

  # --no-ff可以保存之前的分支历史， git merge 不会显示feature，只保留单条分支记录。
  git merge
  git merge --no-ff
  git merge develop --squash  #develop分支的commit message合并为一条

  vim xxx   #手动更改冲突的地方
  git commit -s       #add commit message
  git push origin master

git branch --set-upstream=origin/xxx xxx  #本地分支关联远程分支
git pull --rebase #更新远程分支到本地，如果有冲突，git自动创建分支
git reset HEAD  #撤销commit
git log --graph   #查看分支情况
git log --pretty=oneline  #显示完整commit id
git blame -L num xx.c   #查找xx.c中第num行的提交

git cherry-pic xxx

submoudle:
  git submoudle add repository_add xxx  # add new submodule
  vim .gitmodules # delete related submudle info
  git rm --cached # delete related files
  git submodule update --init --recursive # download submodules
```

# GIT COMMIT 格式
## 格式化作用
1. 提供更多历史信息，方便快速浏览：`git log --pretty=format:%s` （仅显示首行）
2. 可以过滤某些commit（比如文档改动），便于快速查找消息。
3. 可以从commit生成 change log

## COMMIT MESSAGE 格式
- commit message 包含三个部分：header body footer
```shell
<type>(<scope>): <subject>// 空一行<body>// 空一行<footer>
```
header是必须的，body 和 footer可以省略。任何一行不得超过72个字符

#### HEADER
> header 只有一行， type(必须), scope(可选), subject(必须)

- type: 用于说明commit类型:

  |type|description|
  |:---|---|
  |feat|新功能|
  |fix|修补bug|
  |docs|文档|
  |style|格式(不影响代码运行的变动)|
  |refactor|重构(即不是新增功能，也不是修改bug的代码变动)|
  |test|增加测试|
  |chore|构建过程或辅助工具的变动|

  如果type为feat和fix，则该 commit 将肯定出现在 Change log 之中。其他情况（docs、chore、style、refactor、test）由你决定，要不要放入 Change log，建议是不要。

- scope: 用于说明commit影响的范围，比如数据层、控制层、视图层等
- subject: commit 目的的简短描述，不超过50个字符。
  - 以动词开头，使用第一人称现在时，比如change，而不是changed或changes
  - 第一个字母小写
  - 结尾不加句号（.）

#### BODY
> Body 部分是对本次 commit 的详细描述，可以分成多行。

1. 使用第一人称现在时，比如使用change而不是changed或changes。
2. 应该说明代码变动的动机，以及与以前行为的对比。

#### FOOTER
> 只用于两种情况：

1. 不兼容变动
  - 如果当前代码与上一个版本不兼容，则 Footer 部分以BREAKING CHANGE开头，后面是对变动的描述、以及变动理由和迁移方法。
  ```shell
  BREAKING CHANGE: isolate scope bindings definition has changed.
    To migrate the code follow the example below:
    Before:
    scope: {
      myAttr: 'attribute',
    }
    After:
    scope: {
      myAttr: '@',
    }
    The removed `inject` was not generally useful for directives so there should
  ```

2. Revert
  如果当前 commit 用于撤销以前的 commit，则必须以revert:开头，后面跟着被撤销 Commit 的 Header。
  ```shell
  revert: feat(pencil): add 'graphiteWidth' option
  This reverts commit 667ecc1654a317a13331b17617d973392f415f02.
  ```
  - Body部分的格式是固定的，必须写成This reverts commit &lt;hash>.，其中的hash是被撤销 commit 的 SHA 标识符。

  - 如果当前 commit 与被撤销的 commit，在同一个发布（release）里面，那么它们都不会出现在 Change log 里面。如果两者在不同的发布，那么当前 commit，会出现在 Change log 的Reverts小标题下面。
