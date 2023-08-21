# function 1
1. dump volume
    - 查看实例对应的volume id
    - ceph 中查找：rados -p openstack-volumes ls | grep xxx
        - rbd_id.volume-dc5f926b-8066-49c5-9f08-9ae3e4318eee
    - rbd -p openstack-volumes export volume-dc5f926b-8066-49c5-9f08-9ae3e4318eee pvedisk.raw

2. pve import
    - new vm
    - delete disk and CD device
    - import disk
    - qm importdisk <vm-id> pvedisk.raw <storage-local> --format <disk-format>
    - qm importdisk 100 pvedisk.raw local-lvm --format raw
    - change boot sequence at vm options

# function 2
1. build RBD file
    - 查看实例对应的volume id
    - ceph 中查找：rados -p openstack-volumes ls | grep xxx
        - rbd_id.volume-dc5f926b-8066-49c5-9f08-9ae3e4318eee
    - ceph dashboard 中copy from this image to the PVE used pool
2. PVE import
    - new vm
    - ceph dashboard 中, 删除该虚拟机对应的rdb file, 上一步copy的rename为刚删除的rdb file。
    - start vm
