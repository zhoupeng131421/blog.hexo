# poweroff cluster

- stop all data flow in admin node 
```txt
ceph osd set noout
ceph osd set norecover
ceph osd set norebalance
ceph osd set nobackfill
ceph osd set nodown
ceph osd set pause
```

- close OSD nodes
- close monitor nodes

# poweron cluster

- open monitor nodes
- open OSD nodes
- start all data flow
```txt
ceph osd unset noout
ceph osd unset norecover
ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset nodown
ceph osd unset pause
```
