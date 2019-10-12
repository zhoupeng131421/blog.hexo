---
title: hexo with jenkins
tags: jenkins
tags: hexo
tags: nginx
tags: github
categories: 服务配置
---

Hexo没有后台管理程序，网络上基本上都是讲得通过 depoly 的，换台电脑如果没有hexo的node环境，就得远程服务器vim更改，甚是麻烦。随用Jenkins自动下载GitHub中hexo所有文件。

# I. Jenkins install

## 1.1 updata git
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

## 1.2 java 环境安装
> 因为Jenkins依赖Java环境，所以必须安装Java环境

```shell
sudo yum list | grep java   #列出来支持的Java版本
sudo yum install java-1.8.0-openjdk.x86_64  #我选择了这个版本
java -version
```

## 1.3 Jenkins download & install
```shell
yum install maven
mvn -v

wget http://ftp-chi.osuosl.org/pub/jenkins/redhat/jenkins-2.190-1.1.noarch.rpm  # download
sudo yum install jenkins-2.190-1.1.noarch.rpm # install
sudo vim /etc/sysconfig/jenkins # config default server port
  JENKINS_PORT = "8000"
sudo service jenkins start  # start server
```
- 远程登陆 IP:8000 进行初始化设置。
- 插件设置-高级-升级站点，URL改为：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/2.190/update-center.json`
- install git plugin
- install github plugin
- install Maven Integration
- install pipeline maven integration
- install github Authentication plugin
- install github Issues plugin
- instlal github sqs build plugin

# II. Hexo install
## 2.1 nodejs install
``` shell
sudo yum install nodejs
sudo yum install npm
```

## 2.2 hexo install
```shell
npm install hexo-cli -g

sudo mkdir /var/www/html/hexo
sudo chown -R benjo:benjo hexo/
cd hexo
hexo init
npm install
git clone https://github.com/iissnan/hexo-theme-next themes/next #donwnload theme
vim _config.yml:
  theme: next

hexo g # generate static file
hexo s/server # start a server

hexo clean
hexo g
hexo c
```

# III. nginx install
```shell
sodu yum install nginx

sudo systemctl stop httpd   # stop apache server
sudo systemctl disable httpd

sudo vim /etc/nginx/nginx.conf  # config nginx
  root /var/www/html/hexo;
  server_name blog.benjo.top;

sudo systemctl enable nginx   # 开机启动nginx server
sudo systemctl start nginx
```

# IV. jenkins and github config

## 4.1 github config

### 4.1.1 generate token
- github -> setting -> developer setting -> generate new token
- 权限全选，然后生成secret text，并保存.

### 4.1.2 webhooks config
- 进入指定的项目仓库 -> setting -> WebHooks&Service -> add webhook
- 填写jenkins服务器地址: `http://ip:port/github-webhook/`
- 选中 `just the push event` and `Active`

## 4.2 jenkins config

### 4.2.1 global config
- manage jenkins -> configure system -> GitHub 服务器
- 添加凭据，Kind:`Secret text`，Secret:`保存的github token secret text`

### 4.2.2 tools config
- manage jenkins -> global tool configuration ->
- jdk:
```shell
Name: jdk8
JAVA_HOME: /usr/lib/jvm/java-1.8.0-openjdk/
```
- git:
```shell
Name: Default
Path to Git executable: /usr/local/git/bin/git	#根据实际安装位置配置
```
- maven:
```shell
Name: maven
MAVEN_HOME: /usr/share/maven
```

### 4.2.3 new Item
> 项目类型freestyle

- General -> GitHub项目: `https://github.com/xxx/xxx`
- Source code -> Git:
```shell
URL: https://github.com/xxx/xxx.git
Credentials: 新建一个username and password类型的凭据(个人GitHub账户)
```
- Build Triggers: 选中GitHub hook trigger for GITScm Polling
- Build -> Execute shell:
```shell
#!/bin/bash -ilex
cd /var/www/html/hexo/source/_posts/
git pull
cd ../../
hexo clean
hexo g -d
```
首先在hexo source下git clone github 仓库，并创建分支；然后`sudo chown -R jenkins:jenkins /var/www/html/hexo/`，更改权限。
- 保存后，尝试Build Now
