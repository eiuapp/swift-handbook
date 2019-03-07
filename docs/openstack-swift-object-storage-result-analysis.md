

当一个文件上传后，存储文件的结果现象分析

## env

- 系统：Ubuntu Server 16.04×64 
- 存储设置：4T
- 架构部署: 

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| keystone |  192.168.100.50 | keystone node |
| Proxy |  192.168.100.50/192.168.0.50 | 代理节点  |
| object1      |     192.168.100.105    |     存储节点1(zone1)| 
| object2      |      192.168.100.106   |     存储节点2(zone1)| 
| object3      |      192.168.100.107    |     存储节点3(zone1)| 
| object4      |     192.168.100.107     |     存储节点4(zone1)|  



## step

### 理论

以对象为例：在objects目录下存放的是各个partition目录，其中每个partition目录是由若干个suffix_path名的目录和一个hashes.pkl文件组成，suffix_path目录下是由object的hash_path名构成的目录，在hash_path目录下存放了关于object的数据和元数据，因此可以以partition目录为单位将整体复制到新分配的节点磁盘。

### 观察现场

```shell
root@swift105:/srv/node/sdb# ls
accounts  async_pending  containers  objects  tmp
root@swift105:/srv/node/sdb# cd objects/
root@swift105:/srv/node/sdb/objects# ls
0     128  162  185  20   206  24   257  28   313  33   342  360  38   411  453  491  516  53   555  579  586  624  67   760  848  874  90   915  921  97   981  99   999
1016  130  174  19   200  221  242  264  284  315  330  351  361  392  431  462  492  52   534  562  58   596  636  684  778  868  883  908  92   937  974  987  991  auditor_status_ALL.json
1022  149  18   193  201  226  253  266  309  32   340  354  379  408  446  49   507  524  542  576  582  614  66   706  782  87   899  911  920  952  977  989  992  auditor_status_ALL.json
root@swift105:/srv/node/sdb/objects# 
```

### upload

因为现在的文件少，所以，我们寄希望于，我们上传一个新的文件，创建新的 partition ，然后，我们观察一下，这个新的partition中有什么.

通过 chrome: http://192.168.0.50/horizon/project/containers/container/test4 上传一个 object-a.txt 文件

这个时候，先记录一下，文件的头尾，用于等会比较。

```shell
DELL@DESKTOP-MQ9CENU MINGW64 ~/Downloads
$ head object-a.txt && tail object-a.txt
[DEFAULT]
bind_ip = 192.168.100.50
bind_port = 6200
# bind_timeout = 30
# backlog = 4096
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
# disable_fallocate = false
# dump_timestamp = false
#
# This is the path of the URL to access the mini web UI.
# path = /__profile__
#
# Clear the data when the wsgi server shutdown.
# flush_at_shutdown = false
#
# unwind the iterator of applications
# unwind = false

DELL@DESKTOP-MQ9CENU MINGW64 ~/Downloads
$
```
### ring 会产生 partition

如何生成 partition 的呢，请看 [这里](/jwl/openstack-swift-analysis-partition/) 或者 [这里](/post/openstack-swift-analysis-partition/) 

### 观察现场

默认配置下，会产生partition，并复制3份，我在 100.105，100.106，100.107-sdb 中都找到了（107-sdc 中没有）。下面我就以 100.105 为例，进去分析：

```shell
root@swift105:/srv/node/sdb/objects# ls
0     119  149  18   193  201  226  253  266  309  32   340  354  379  408  446  49   507  524  542  576  582  614  66   706  782  87   899  911  920  952  977  989  992                      auditor_status_ALL.json
1016  128  162  185  20   206  24   257  28   313  33   342  360  38   411  453  491  516  53   555  579  586  624  67   760  848  874  90   915  921  97   981  99   999
1022  130  174  19   200  221  242  264  284  315  330  351  361  392  431  462  492  52   534  562  58   596  636  684  778  868  883  908  92   937  974  987  991  auditor_status_ALL.json
root@swift105:/srv/node/sdb/objects# 
```

### partition目录 - 119

观察到，多出了一个 119 文件夹，这个 119 就是我们的 object partition了。
```shell
root@swift105:/srv/node/sdb/objects# cd 119 && ls 
611  hashes.invalid  hashes.pkl
root@swift105:/srv/node/sdb/objects/119# 
``` 

- hashes.invalid ： 这个是什么，我不知道，谁能告诉我
- 611 : suffix_path名的目录
- hashes.pkl ： a pickled dictionary.

#### hashes.pkl 

The file itself is a pickled dictionary. The dictionary contains a dictionary where the key is a suffix directory name and the value is the MD5 hash of the directory listing for that suffix. In this manner, the daemon can quickly identify differences between local and remote suffix directories on a per partition basis as the scope of any one hashes.pkl file is a partition directory.

具体看  https://docs.openstack.org/swift/queens/overview_replication.html#hashes-pkl

```shell
root@swift105:/srv/node/sdb/objects/119# cp hashes.pkl ~/a.pkl && cd && ls
a.pkl
root@swift105:~# python
Python 2.7.12 (default, Nov 12 2018, 14:36:49) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import cPickle as pickle
>>> fr = open('a.pkl')
>>> inf = pickle.load(fr)
>>> print(inf)
{u'611': 'e1955615281c88159dcc46d12766c6ef'}
>>> fr.close()
>>> exit()
root@swift105:~# 
``` 

#### hashes.invalid 

这个是什么，我不知道，谁知道的？

### suffix_path目录 - 611

suffix_path目录下是由object的hash_path名构成的目录

```shell
root@swift105:/srv/node/sdb/objects/119# cd 611/ && ls
1df3d15f3a9a2b788e373576bcf97611
root@swift105:/srv/node/sdb/objects/119/611# 
```

最后3位是 '611'

### object的hash_path目录 - 1df3d15f3a9a2b788e373576bcf97611

在hash_path目录下存放了关于object的数据和元数据

#### object的数据 - object-a.txt
```shell
root@swift105:/srv/node/sdb/objects/119/611# cd 1df3d15f3a9a2b788e373576bcf97611/ && ls
1548756536.41498.data
root@swift105:/srv/node/sdb/objects/119/611/1df3d15f3a9a2b788e373576bcf97611# 
```

同时，我们可以看一下这个数据的数据内容。

```shell
root@swift105:/srv/node/sdb/objects/119/611/1df3d15f3a9a2b788e373576bcf97611# cp 1548756536.41498.data ~/a.txt && head ~/a.txt && tail ~/a.txt
[DEFAULT]
bind_ip = 192.168.100.50
bind_port = 6200
# bind_timeout = 30
# backlog = 4096
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
# disable_fallocate = false
# dump_timestamp = false
#
# This is the path of the URL to access the mini web UI.
# path = /__profile__
#
# Clear the data when the wsgi server shutdown.
# flush_at_shutdown = false
#
# unwind the iterator of applications
# unwind = false
root@swift105:/srv/node/sdb/objects/119/611/1df3d15f3a9a2b788e373576bcf97611# 
```

与之前的内容比较，知道，就是这个文件，文件内容一样。

#### object的元数据 

元数据存储在文件系统的扩展属性（xattr）中。

那怎么才能看得到呢？


## ref

- https://docs.openstack.org/swift/queens/overview_replication.html#hashes-pkl