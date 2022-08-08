---
title: PVE config iommu
date: 2022-06-12
tags: [iommu, PVE]
categories: 服务配置
---

1. vim /etc/default/grub
```shell
#GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```

- update-grub
2. vim /etc/modules
```shell
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

- update-initramfs -u -k all
