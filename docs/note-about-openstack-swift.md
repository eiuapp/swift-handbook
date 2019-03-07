+++
title = "openstack swift(笔记)"
date = 2019-01-19T00:00:00-08:00
lastmod = 2019-01-21T23:40:11-08:00
tags = ["swift"]
categories = ["swift"]
draft = false
weight = 3001
+++

## replication 默认为3 {#replication-默认为3}

每次上传的文件，会经过ring选择3个硬盘存储。

可以看下面这个图的倒数3行命令的结果

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1547888197/swift-upload-file-only-storage-three-hardware%5Fktkven.png)


## eventually consistent {#eventually-consistent}

Eventually Consistent：最终一致性 看一下 <http://duanple.blog.163.com/blog/static/7097176720114733211308/>


## rsyncd做同步 {#rsyncd做同步}

当增删storage node，导致各个节点的数据不均衡时，rsync会自动去均衡。

可以通过查看 rsync 的日志 /var/log/rsyncd.log

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1548142119/swift-rsync-check%5Fno10ug.png)


## swift 的简单介绍视频 {#swift-的简单介绍视频}

关于一个 swift 的简单介绍视频，很不错的。地址在：<https://pan.baidu.com/s/16KFxATE8CG%5FEDkIlWqdURg>

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1547888197/storage%5Fxzqbit.png)

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1547888197/swift-tedian%5Fxqdlsl.png)

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1547888197/swift-tools%5Fwydi8n.png)

![figure](https://res.cloudinary.com/dmtixvmgt/image/upload/v1547888197/swift-usual-wrong-code%5Fp36yjd.png)
