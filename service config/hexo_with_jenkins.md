Hexo没有后台管理程序，网络上基本上都是讲得通过 depoly 的，换台电脑如果没有hexo的node环境，就得远程服务器vim更改，甚是麻烦。随用Jenkins自动下载GitHub中hexo所有文件。

# Summary
- Jenkins java install & config
- nodejs hexo install & config
- nginx install & config

# Jenkins install
## java 环境安装
> 因为Jenkins依赖Java环境，所以必须安装Java环境

```shell
sudo yum list | grep java   #列出来支持的Java版本
sudo yum install java-1.8.0-openjdk.x86_64  #我选择了这个版本
java -version
```

## Jenkins download & install
```shell
wget http://ftp-chi.osuosl.org/pub/jenkins/redhat/jenkins-2.190-1.1.noarch.rpm  # download
sudo yum install jenkins-2.190-1.1.noarch.rpm # install
sudo vim /etc/sysconfig/jenkins # config default server port
  JENKINS_PORT = "8000"
sudo service jenkins start  # start server
```
- 远程登陆 IP:8000 进行初始化设置。
- 插件设置-高级-升级站点，URL改为：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/2.190/update-center.json`
- install git pulgin

# Hexo install
## nodejs install
``` shell
sudo yum install nodejs
sudo yum install npm
```

## hexo install
```shell
npm install hexo-cli -g

sudo mkdir /var/www/html/hexo
sudo chown -R benjo:benjo hexo/
cd hexo
npm install
hexo g # generate static file
hexo s/server # start a server
```

# nginx install
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

