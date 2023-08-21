---
title: Gentoo install on PVE
date: 2022-10-26
tags: [Gentoo]
categories: 随笔
---

# Prepare
- liveCD:
    - [install-amd64-minimal.iso](https://mirror.bytemark.co.uk/gentoo//releases/amd64/autobuilds/20221023T170534Z/install-amd64-minimal-20221023T170534Z.iso)
- stage:
    - [stage3-amd64-desktop-systemd.tar.xz](https://mirrors.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-systemd/stage3-amd64-desktop-systemd-20221023T170534Z.tar.xz)
- mirror:
    - [mirrors.tuna](https://mirrors.tuna.tsinghua.edu.cn/gentoo)
- PVE create VM
    - BIOS: q35 OVMF (UEFI)
    - CPU: Host (Intel(R) Xeon(R) Gold 5122)
    - Memory: ballooning device
    - SCIS Controller: VirtIO SCSI
    - Network Device: VirtIO (paravirtualized)




# Install
- BIOS
    - disable secure boot
- sshd config
    - rc-service sshd start
    - passwd
- get ip address via `ip addr`

- login vm via ssh
    - ssh root@x.x.x.x
- format disk
```shell
fdisk /dev/sda
    g
    n
    1
    +256M
    n
    2
    +8G
    n
    3

mkfs.ext4 /dev/sda3
mkfs.fat -F32 /dev/sda1
mkswap /dev/sda2
swapon /dev/sda2

fdisk -l
    Disk /dev/sda: 64 GiB, 68719476736 bytes, 134217728 sectors
    Disk model: QEMU HARDDISK
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: gpt
    Disk identifier: 27F337EC-98F5-3B43-9285-B1F067689164

    Device        Start       End   Sectors  Size Type
    /dev/sda1      2048    526335    524288  256M Linux filesystem
    /dev/sda2    526336  17303551  16777216    8G Linux filesystem
    /dev/sda3  17303552 134217694 116914143 55.7G Linux filesystem
```
- mount /dev/sda3 /mnt/gentoo
- sync time: ntpd -q -g
### install stage3
- cd /mnt/gentoo
- wget https://mirrors.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-systemd/stage3-amd64-desktop-systemd-20221023T170534Z.tar.xz
- tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
- config compile paramete: nano etc/portage/make.conf:
```shell
COMMON_FLAGS="-O2 -pipe" -> COMMON_FLAGS="-march=native -O2 -pipe"

# fix "no-source-code license(s), missing keyword"
ACCEPT_LICENSE="*"

# count for cpus
MAKEOPTS="-j12"
```
- set mirror
```shell
echo 'GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo"' >> /mnt/gentoo/etc/portage/make.conf
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```
- mount fs
```shell
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```
- switch to new envirnoment:
```shell
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```
- mount /dev/sda1 /boot
- config portage
```shell
emerge-webrsync
eselect profile list
    - [1]   default/linux/amd64/17.1 (stable)
    ...
    - [7]   default/linux/amd64/17.1/desktop/gnome/systemd (stable)
    - [12]  default/linux/amd64/17.1/desktop/systemd (stable) *
eselect profile set 7
```
- update: emerge --ask --verbose --update --deep --newuse @world
- config time
```shell
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'zh_CN.UTF-8 UTF-8' >> /etc/locale.gen
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
locale-gen

eselect locale list
eselect locale set 2
```
- env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
- emerge --ask vim networkmanager
## build kernel
- option：emerge --ask sys-kernel/linux-firmware
- emerge --ask sys-kernel/gentoo-sources
- emerge --ask sys-kernel/genkernel
- vim etc/fstab
```shell
/dev/sda1       /boot   vfat    defaults,noatime 0       2
/dev/sda2       swap    swap    defaults         0       0
/dev/sda3       /       ext4    noatime 0        1
```
- config build
    - emerge --ask sys-apps/pciutils
        - find all pci device
        - or use `lsmod`
    - cd /usr/src/linux
    - make menuconfig
    - make && make modules_install
    - make install
    - emerge --ask sys-kernel/dracut
    - dracut --kver=5.15.75-gentoo
    - ls /boot/initramfs*
- genkernel
    - emerge --ask sys-kernel/genkernel
    - nano /etc/fstab: /boot
    - genkernel --virtio all
    - ls /boot/
- distribution kernel
    - emerge --ask sys-kernel/installkernel-systemd-boot
    - emerge --ask sys-kernel/gentoo-kernel(-bin)
    - emerge --ask @module-rebuild

### config sys
- echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
- emerge --ask sys-boot/grub

- grub-install --target=x86_64-efi --efi-directory=/boot
- grub-mkconfig -o /boot/grub/grub.cfg

### end
- exit
- cdimage ~#cd
- cdimage ~#umount -l /mnt/gentoo/dev{/shm,/pts,}
- cdimage ~#umount -R /mnt/gentoo
- cdimage ~#reboot

### xfce4
- use GDM as display manager
    - `emerge --ask gnome-base/gdm`
    - `systemctl enable gdm`
    - `systemctl start gdm`
- `emerge --ask --oneshot xfce-extra/xfce4-notifyd`
- `emerge --ask xfce-base/xfce4-meta`
    - test: startxfce4
- `echo XSESSION=\"Xfce4\" > /etc/env.d/90xsession`
- `env-update && source /etc/profile`


- 
emerge --ask vim networkmanager net-misc/chrony net-misc/dhcpcd
emerge --ask sys-kernel/linux-firmware sys-kernel/gentoo-sources sys-kernel/genkernel
emerge --ask sys-kernel/dracut

echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
emerge --ask sys-boot/grub

- 
passwd
systemd-firstboot --prompt --setup-machine-id
systemctl preset-all
systemctl enable sshd
systemctl enable chronyd
systemctl enable dhcpcd





mount /dev/sda3 /mnt/gentoo
ntpd -q -g
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
mount /dev/sda1 /boot