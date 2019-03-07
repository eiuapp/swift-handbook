+++
title = "swift storage node 出现 xfs 错误"
date = 2019-01-09T00:00:00+08:00
lastmod = 2019-01-09T15:14:12+08:00
tags = ["openstack", "swift", "xfs"]
categories = ["openstack"]
draft = false
weight = 3002
+++

## env {#env}

在安装 swift storage node 过程中，把storage node关机，再开机时，开机登陆页面，出现如下的错误：

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1547016078/xfs-internal-error-xfs%5Fwant%5Fcorrupted%5Freturn%5Fe77thd.jpg)

错误中，主要是包含 "XFS(sdb): Internal error xfs\_trans\_cancel", "XFS(sdb):
Corruption of in -memory data detected" 等字样，这明显是 xfs 相关的问题。

在安装过程中，与 xfs 相关，主要就是挂载分区或挂载硬盘的时候咯。

而且查看 storage node 的挂载文件夹

```shell
root@swift107:/etc/swift# chown -R swift:swift /srv/node
chown: cannot access '/srv/node/sdb': Input/output error
root@swift107:/etc/swift# cd /srv/node/
root@swift107:/srv/node# ls
ls: cannot access 'sdb': Input/output error
sdb  sdc
root@swift107:/srv/node#
```

而且，如果是在 controller node 有监控的话，可以看到：

```shell
root@controller:/etc/swift# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Wed 2019-01-09 10:19:46 CST; 5min ago
     Docs: man:systemd-sysv-generator(8)
  Process: 5475 ExecStop=/etc/init.d/swift-proxy stop (code=exited, status=0/SUCCESS)
  Process: 5486 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 100.3M
      CPU: 5.790s
   CGroup: /system.slice/swift-proxy.service
           ├─5497 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─5506 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─5507 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─5508 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─5509 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Jan 09 10:23:39 controller proxy-server[5509]: ERROR Insufficient Storage 192.168.100.107:6200/sda6 (txn: txff3a41a6156d41b39866c-00
Jan 09 10:23:39 controller proxy-server[5509]: ERROR Insufficient Storage 192.168.100.107:6200/sdb (txn: txff3a41a6156d41b39866c-005
Jan 09 10:23:39 controller proxy-server[5509]: 127.0.0.1 127.0.0.1 09/Jan/2019/02/23/39 PUT /v1/AUTH_7d6eaa90d74a4f239963933c3a744df
Jan 09 10:23:39 controller proxy-server[5508]: 127.0.0.1 127.0.0.1 09/Jan/2019/02/23/39 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df
Jan 09 10:23:39 controller proxy-server[5507]: ERROR Insufficient Storage 192.168.100.107:6201/sdb (txn: txba2112e7ac2c4fd09ebc7-005
Jan 09 10:23:39 controller proxy-server[5507]: 127.0.0.1 127.0.0.1 09/Jan/2019/02/23/39 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df
root@controller:/etc/swift#
```

报出了 " ERROR Insufficient Storage" 错误。


## step {#step}

通过 - <https://www.experts-exchange.com/questions/26974279/Problem-with-xfs-file-system.html>
知道，要先 umount 再使用 \`xfs\_repair -L \`去修复一下，再mount。

```shell
root@swift107:/home/ubuntu# umount /srv/node/sdb
umount: /srv/node/sdb: not mounted
root@swift107:/home/ubuntu# umount /srv/node/sdc
root@swift107:/home/ubuntu# xfs_repair -L /dev/sdb
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ALERT: The filesystem has valuable metadata changes in a log which is being
destroyed because the -L option was used.
        - scan filesystem freespace and inode maps...
sb_ifree 199, counted 159
sb_fdblocks 5237633, counted 5196324
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
correcting nblocks for inode 33575016, was 1 - counted 0
imap claims a free inode 33575022 is in use, correcting imap and clearing inode
cleared inode 33575022
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 1
        - agno = 3
        - agno = 0
        - agno = 2
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
entry "hashes.pkl" in directory inode 33575010 references already connected inode 33575016.

        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
Maximum metadata LSN (4:8073) is ahead of log (1:2).
Format log to cycle 7.
done
root@swift107:/home/ubuntu# xfs_repair -L /dev/sda6
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 2
        - agno = 3
        - agno = 1
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
Maximum metadata LSN (1:30) is ahead of log (1:2).
Format log to cycle 4.
done
root@swift107:/home/ubuntu# mount /srv/node/sdb
root@swift107:/home/ubuntu# ls /srv/node/sdb -a
.  ..  accounts  containers  objects  quarantined  tmp
root@swift107:/home/ubuntu# ls /srv/node/sdc -a
.  ..
root@swift107:/home/ubuntu# mount | grep sd
/dev/sda1 on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/sdb on /srv/node/sdb type xfs (rw,noatime,nodiratime,attr2,nobarrier,inode64,logbufs=8,noquota)
root@swift107:/home/ubuntu# du -sh /srv/node/sdb
291M	/srv/node/sdb

```

然后，把 controller node 中的 swift-proxy 重启一下。

```shell
service swift-proxy restart
service swift-proxy status
```

再上传文件，看一下 /srv/node/sdb 的大小是否有增加。

```shell
root@swift107:/home/ubuntu# du -sh /srv/node/sdb
498M	/srv/node/sdb
```

因为这台机器的 /dev/sda6 分区，也是做成了  xfs 类型文件系统，所以，一样地处理一
下。

```shell
root@swift107:/home/ubuntu# mount /srv/node/sdc
mount: /dev/sda6 is already mounted or /srv/node/sdc busy
       /dev/sda6 is already mounted on /srv/node/sdc
root@swift107:/home/ubuntu# mount | grep sd
/dev/sda1 on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/sdb on /srv/node/sdb type xfs (rw,noatime,nodiratime,attr2,nobarrier,inode64,logbufs=8,noquota)
/dev/sda6 on /srv/node/sdc type xfs (rw,noatime,nodiratime,attr2,nobarrier,inode64,logbufs=8,noquota)
root@swift107:/home/ubuntu# du -sh /srv/node/sdc
0	/srv/node/sdc
root@swift107:/home/ubuntu#
```

这样子，就完成了。


## Ref {#ref}

-   <https://www.experts-exchange.com/questions/26974279/Problem-with-xfs-file-system.html>
