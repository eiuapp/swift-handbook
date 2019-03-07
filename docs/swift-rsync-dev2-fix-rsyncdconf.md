

观察 swift storage node 间通过 rsync 进行数据通信。

最后结论：

要把最新的 *.ring.gz 文件分发至所有的 storage node ，然后 节点运行 `swift-init all restart`

## step

修改 /etc/rsyncd.conf

然后，把各节点 rsync 重启
```shell
service rsync restart
service rsync status
```

此时，180,198,135的rsync日志已经出现新的同步信息。

说明，不需要重启启动`swift-init`服务。

但是，127，134的rsync日志，依旧没有变化。(很可能没有开始同步，为什么呢？是需要时间等待么？）

现在发现 134的 `/etc/swift/*.ring.gz` 文件不是rebalance后的*.ring.gz文件，所以，这个时候，我怀疑是不是这个地方的问题。

从51中复制*.ring.gz文件给 134，重启rsync，没有同步。重启swift-init。

```shell
swift-init all status
swift-init all restart
swift-init all status
```

依然没有同步。为什么？

好了。这个时候发现 原来 180，198，135能同步，而134，127与其它3个连接不上，是因为， 180，198，135使用的是同一个老版本的 *.ring.gz 文件。晕死了！

这个时候，从51中复制*.ring.gz文件给 180。不重启任何服务。这时，

127 rsync日志

```text
2019/01/22 17:26:37 [25850] connect from swift0180 (192.168.0.180)
2019/01/22 09:26:37 [25850] rsync to object/sdb/objects/834 from swift0180 (192.168.0.180)
2019/01/22 09:26:37 [25850] receiving file list
2019/01/22 09:26:44 [25850] sent 62 bytes  received 74104148 bytes  total size 74085496
```

134 rsync日志

```text
2019/01/22 17:26:43 [6716] connect from swift0180 (192.168.0.180)
2019/01/22 17:26:43 [6717] connect from swift0180 (192.168.0.180)
2019/01/22 09:26:43 [6716] rsync to account/sdb/tmp/a3ed88a4-5dcb-4f09-9122-96c590ec12cf from swift0180 (192.168.0.180)
2019/01/22 09:26:43 [6716] receiving file list
2019/01/22 09:26:43 [6717] rsync to account/sdb/tmp/2043b019-6d91-4c89-b1c0-ab4a9b05f460 from swift0180 (192.168.0.180)
2019/01/22 09:26:43 [6717] receiving file list
2019/01/22 17:26:45 [6720] connect from swift0180 (192.168.0.180)
2019/01/22 09:26:45 [6720] rsync to object/sdb/objects/538 from swift0180 (192.168.0.180)
2019/01/22 09:26:45 [6720] receiving file list
2019/01/22 17:27:00 [6723] connect from swift0180 (192.168.0.180)
2019/01/22 17:27:00 [6724] connect from swift0180 (192.168.0.180)
2019/01/22 09:27:00 [6723] rsync to container/sdb/tmp/42bc9ff4-1ad1-4802-bfbc-40796ca93fde from swift0180 (192.168.0.180)
2019/01/22 09:27:00 [6723] receiving file list
2019/01/22 09:27:00 [6724] rsync to container/sdb/tmp/0f6aa3dc-2abd-4259-a0f3-89ad321882c1 from swift0180 (192.168.0.180)
2019/01/22 09:27:00 [6724] receiving file list
```

但是，都停止在这里了，没有下文了。然后，我再回 180看 rsync日志
都是与 135,198 的同步信息。

我考虑把 180 rsync 重启。

无效，依然连接的是 135，198.

180 
```
swift-init restart 
```

134

```
2019/01/22 17:34:46 [6768] connect from swift0180 (192.168.0.180)
2019/01/22 09:34:46 [6768] rsync to object/sdb/objects/415 from swift0180 (192.168.0.180)
2019/01/22 09:34:46 [6768] receiving file list
2019/01/22 17:34:47 [6770] connect from swift0180 (192.168.0.180)
2019/01/22 17:34:47 [6770] max connections (2) reached
2019/01/22 17:34:47 [6771] connect from swift0180 (192.168.0.180)
2019/01/22 17:34:47 [6771] max connections (2) reached
2019/01/22 17:34:49 [6772] connect from swift0180 (192.168.0.180)
2019/01/22 17:34:50 [6772] max connections (2) reached
2019/01/22 17:34:50 [6773] connect from swift0180 (192.168.0.180)
2019/01/22 17:34:50 [6773] max connections (2) reached
2019/01/22 17:35:17 [6779] connect from swift0180 (192.168.0.180)
2019/01/22 17:35:17 [6779] max connections (2) reached
```


127
```text
2019/01/22 09:34:45 [25913] rsync to object/sdb/objects/415 from swift0180 (192.168.0.180)
2019/01/22 09:34:45 [25913] receiving file list
2019/01/22 09:34:46 [25913] sent 31 bytes  received 198 bytes  total size 487805144
2019/01/22 17:34:47 [25935] connect from swift0180 (192.168.0.180)
2019/01/22 09:34:47 [25935] rsync to container/sdb/tmp/0f6aa3dc-2abd-4259-a0f3-89ad321882c1 from swift0180 (192.168.0.180)
2019/01/22 09:34:47 [25935] receiving file list
2019/01/22 09:34:48 [25935] sent 44 bytes  received 21625 bytes  total size 21504
```

无效，依然连接的是 135，198.

这个说明，这个同步机制，会由proxy协调，并，不受 本地的rsync 影响。

现在这个时候，chrome访问： http://192.168.0.51/horizon/auth/login/?next=/horizon/project/

这个时候，看来，只能是把 *.ring.gz 文件分发给所有节点，再试一下了。

----------------

同时，观察180的 /var/log/rsync.log 

```text
2019/01/22 09:10:24 [22897] receiving file list
2019/01/22 09:10:24 [22897] rsync: mkdir "/sdc/objects/832" (in object) failed: No such file or directory (2)
2019/01/22 09:10:24 [22897] rsync error: error in file IO (code 11) at main.c(674) [Receiver=3.1.1]
```
这里出现了 `rsync error: error in file IO (code 11) at main.c(674) [Receiver=3.1.1]`，这个是指同步出错了么？

-----------------

解决完这个问题后，发现，最后，依然是要把最新的 *.ring.gz 文件分发至所有的 storage node ，然后 节点运行 `swift-init all restart`