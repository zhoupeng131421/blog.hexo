---
title: linux install NVIDIA driver
date: 2021-11-4
tags: [linux]
categories: 随笔
---

# prepare
- sudo yum update -y
- sudo yum install kernel-devel gcc -y
	- 确保kernel version 一致

- lsmod | grep nouveau
- sudo vim /lib/modprobe.d/dist-blacklist.conf
```shell
	#blacklist nvidiafb 
	# add
	blacklist nouveau
	options nouveau modeset=0
```
- sudo mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
- sudo dracut /boot/initramfs-$(uname -r).img $(uname -r)
- sudo systemctl set-default multi-user.target
- sudo reboot

# install
- download drvier from NIVIDIA offical web
- sudo systemctl stop gdm
- sudo ./NVIDIA-Linux-x86_64-470.82.00.run --kernel-source-path=/usr/src/kernels/3.10.0-1160.45.1.el7.x86_64 -k $(uname -r)
    - the path of kernel-source depends your real environment
- sudo systemctl start gdm
- nvidia-smi
