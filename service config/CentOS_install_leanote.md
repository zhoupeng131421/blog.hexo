---
title: Centos install leanote
tags: [Centos, leanote]
categories: 服务配置
---

# centos_install_leanote

### install mongodb
> 因leanote需要数据库管理数据，所以选择mongodb（也可以选择sql)

```
yum update
yum upgrade
yum install mongodb mongond-server
# start server
service mongod start
```

### download leanote
```
mkdir /usr/local/leanote/
cd /usr/local/leanote/
wget https://nchc.dl.sourceforge.net/project/leanote-bin/2.6.1/leanote-linux-amd64-v2.6.1.bin.tar.gz
tar xvzf leanote-linux-amd64-v2.6.1.bin.tar.gz
cd leanote

# import datebase
mongorestore -h localhost -d leanote --dir mongodb_backup/leanote_install_data/
```

### install OpenResty
> 升级版的nginx

```
yum install readline-devel pcre-devel openssl-devel -y

# download
wget https://openresty.org/download/openresty-1.15.8.2.tar.gz
tar -xvzf openresty-1.15.8.2
cd openresty
./configure
make
make install

# take a ln
ln -s /usr/local/openresty/nginx/sbin/nginx /usr/sbin/nginx
```

- config a server: /usr/local/openresty/nginx/conf/nginx.conf
```
server {
        listen       80;
        server_name  notes.benjo.top;
        charset utf-8;
        location / {
            default_type text/html;
            proxy_pass http://127.0.0.1:9000;
        }
}
```

### install wkhtmltopdf
> export PFD document

```
wget https://codeload.github.com/wkhtmltopdf/wkhtmltopdf/tar.gz/0.12.5
tar -xvzf wkhtmltopdf
cd wkhtmltopdf
chmod +x wkhtmltopdf
cp wkhtmltopdf /usr/local/bin/
```

### start leanote server
```
cd ~/leanote/bin
chmod +x run.sh
./runsh &

# restart nginx
nginx
```
- leanote default account is 'admin', passwd is 'abc123'
