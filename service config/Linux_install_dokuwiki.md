---
title: linux install dokuwiki
tags: [linux, dokuwiki]
categories: 服务配置
---

# Apache2 config
- 配置虚拟主机
`/etc/apache2/siet-available/` 更改默认配置 `000-default.conf`:`DocumentRoot /var/www/dokuwiki`

- 添加端口
/etc/apache2/ports.conf 文件可以用来添加一个新的监听端口
```shell
Listen 80
Listen 8080
```

- restart server: `sudo service apache2 reload`

# 配置dokuwiki
- 下载：
```
cd /var/www
sudo wget https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz
sudo tar -xvzf dokuwiki-stable.tgz
sudo mv dokuwiki-*/ dokuwiki
```
- 更改权限：`sudo chown -R www-data:www-data /var/www/dokuwiki`

# Config dokuwiki
- 浏览器访问`http://IP:/install.php` 进行初始化配置
- 配置完后删除`/var/www/dokuwiki/install.php`
- data/pages/下中文路径乱码修复:
```shell
#:vim conf/local.php
+ $conf['fnencode'] = 'utf-8';
```

### 插件
- Add New Page Plugin
- Move Plugin
- Markdown Page Plugin
- Indexmenu Plugin
- simplenavi Plugin
```
vim sidebar.txt
	{{simplenavi>}}
	{{NEWPAGE}}
```

### 安全配置
- 方法1：修改`/etc/apache2/apache2.conf`,使内外文件不可访问，增加:
```
<Directory /var/www/html/dokuwiki>
        order deny,allow
        allow from all
</Directory>
<LocationMatch "/(data|conf|bin|inc)/">
        order allow,deny
        deny from all
        satisfy all
</LocationMatch>
```
然后重启apache2服务
- 方法2： `000-default.conf` 增加：
```
        <Directory /var/www/html/dokuwiki>
            AllowOverride All
            allow from all
        </Directory>
        <LocationMatch "/(data|conf|bin|inc)/">
            order allow,deny
            deny from all
            satisfy all
        </LocationMatch>
```
