---
title: windows系统下git环境的配置
---

# 安装
- https://git-scm.com/download/win  下载window版本git，并安装。
- 安装：选择Notepad++作为默认编辑器（提前安装Notepad++）,其他默认安装即可。

# 配置github账号信息
```shell
git config --global user.name xxx
git config --global user.email xxx@xxx.xxx

cd ~/.ssh
ssh-keygen.exe -t rsa -C xxx@xxx.com    //生成key
```
- github账户信息中添加SSH key
- 验证账户SSH key, `ssh.exe –T git@github.com`

# git clone
```shell
git clone git@github.com:nieklinnenbank/FreeNOS.git
```

# G++安装
1. http://www.mingw.org/ 下载windows版 MinGW  mingw-get-setup.exe 下载并安装
2. 打开installation manager界面，选中mingw32-gcc-g++-bin,左上角菜单installation->apply change
3. 环境变量设置，Path中添加`C:\MinGW\bin`
4. 测试安装完成：powershell 输入 `g++ -v`看是否成功:
