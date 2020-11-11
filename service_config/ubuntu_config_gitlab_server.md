---
title: gitlab backup
date: 2020-11-10
tags: [gitlab]
categories: 服务配置
---

- 创建备份：gitlab-rake gitlab:backup:create

- 自动挂载iSCSI device: `vim /etc/fstab`:`/dev/sdb /var/opt/gitlab/backups_remote ext4 defaults,_netdev 0 0`
- 改变备份位置: `vim /etc/gitlab/gitlab.rb`
```shell
gitlab_rails['manage_backup_path'] = true
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups_remote"

# ms
gitlab_rails['backup_keep_time'] = 2419200
```
- apply config: `gitlab-ctl reconfigure`

- automatic exec backup: `vim /etc/crontab`:
    - `0  4    * * *   root    /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1`
    - 每天凌晨4点进行备份
    - service cron restart