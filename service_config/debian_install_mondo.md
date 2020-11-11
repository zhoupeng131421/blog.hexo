---
title: debian install mondo
date: 2020-10-12
tags: [debian, mondo]
categories: 服务配置
---

## mondo
- export PATH=$PATH:/sbin
- apt install vim net-tools

- wget ftp://ftp.mondorescue.org/debian/10/mondorescue.sources.list
- sh -c "cat mondorescue.sources.list >> /etc/apt/sources.list"
- apt update
	- there will have failure: "GPG error: The following signatures couldn't be verified because the public key is not available"
	- gpg --keyserver keyserver.ubuntu.com --recv-keys 8B48AD6246925553
	- gpg -a --export 8B48AD6246925553 | apt-key add -
	- apt update
- apt install mondo
- mondoarchive
