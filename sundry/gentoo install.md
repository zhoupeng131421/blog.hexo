- PVE create VM
    - BIOS: OVMF (UEFI)
    - Memory: no ballooning device
    - SCIS Controller: LSI
    - Network Device: vmxnet3
## install
- sshd config
    - rc-service sshd start
    - passwd
        - ssh root@x.x.x.x
- parted: `parted /dev/sda`
```shell
mklabel gpt
mkpart primary 1 3
mkpart primary 3 259
mkpart primary 259 4355
mkpart primary 4355 -1

name 1 bios
name 2 boot
set 1 bios on
set 2 boot on

quit
```
- mkfs:
```shell
mkfs.vfat /dev/sda2
mkfs.ext4 /dev/sda4
mkswap /dev/sda3
swapon /dev/sda3
```
- mount:
```shell
mount /dev/sda4 /mnt/gentoo
mkdir /mnt/gentoo/boot
mount /dev/sda2 /mnt/gentoo/boot
```
### install stage3
- cd /mnt/gentoo
- wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20220710T170538Z/stage3-amd64-desktop-systemd-20220710T170538Z.tar.xz
- tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
- nano etc/portage/make.conf: `MAKEOPTS='-j4'`
- cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
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
- chroot /mnt/gentoo /bin/bash
- source /etc/profile
- export PS1="(chroot) ${PS1}"
- emerge-webrsync
    - emerge --sync
- eselect profile list
- eselect profile set 10
    - [10]  default/linux/amd64/17.1/desktop/systemd (stable) *
- emerge --ask --verbose --update --deep --newuse @world
- config time
```shell
echo Asia/Shanghai > /etc/timezone
emerge --config sys-libs/timezone-data
echo 'zh_CN.UTF-8 UTF-8' >> /etc/locale.gen
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
locale-gen

eselect locale list
eselect locale set 2
```
- env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
- emerge --ask sys-kernel/gentoo-sources
- emerge --ask sys-kernel/genkernel
- vim etc/fstab
```shell
UUID=23c6c36c-af49-4acf-96a2-1cd1a64be15e       /       ext4    noatime 0       1
UUID=28DE-5946  /boot   vfat    default,noatime 0       2
UUID=7b2e7ecc-b78c-49ad-9226-15c5178ed9ea       swap    swap    defaults        0       0
```
- emerge --ask sys-apps/pciutils
- cd /usr/src/linux
- make menuconfig
- make && make modules_install
- make install
- emerge --ask sys-kernel/dracut
- dracut --kver=5.15.52-gentoo
- ls /boot/initramfs*

### config sys
- emerge --ask --verbose sys-boot/grub
- echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
- emerge --ask sys-boot/grub
- emerge --ask --update --newuse --verbose sys-boot/grub
- grub-install --target=x86_64-efi --efi-directory=/boot
- grub-mkconfig -o /boot/grub/grub.cfg

### end
- exit
- cdimage ~#cd
- cdimage ~#umount -l /mnt/gentoo/dev{/shm,/pts,}
- cdimage ~#umount -R /mnt/gentoo
- cdimage ~#reboot