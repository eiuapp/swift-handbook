
如果出现 ` mount : **** Structure needs cleaning` 也是通过 xfs_repair -L 来修复

```
[root@pc4 ~]# mount /src/node/sdb/
mount: Structure needs cleaning

[root@pc4 ~]# xfs_repair -L /dev/sda1 
```
