+++
title = "swift 安装过程（queen）"
date = 2019-01-08T00:00:00+08:00
lastmod = 2019-01-09T16:27:38+08:00
tags = ["openstack", "swift", "install"]
categories = ["openstack"]
draft = false
weight = 3001
+++

主要记录一下根据官方文档安装过程中遇到的一些小的问题。


## env {#env}

os: 均为 ubuntu server 16.04
controller node:

-   192.168.100.50

网络适配器 3 个：

-   VMnet0
-   VMnet1
-   VMnet8(NAT)

storage node:

-   192.168.100.105
-   192.168.100.106
-   192.168.100.107

各节点网络适配器相同，2个：

-   VMnet0
-   VMnet1

注意：钱工说 VMnet0 就是桥接网络

以 storage node 中的 IP 为 192.168.100.107 为例子，展示一下 /etc/hosts 文件 。

```shell
ubuntu@swift107:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.0.1	swift107
127.0.1.1	ubuntu
# controller
192.168.100.50	controller
# storage
192.168.100.105	swift105
192.168.100.106	swift106
192.168.100.107	swift107
```

官方文档这里只显示了 storage node :

-   10.0.0.51       object1
-   10.0.0.52       object2


## step {#step}


### environment networking {#environment-networking}

<https://docs.openstack.org/swift/queens/install/environment-networking.html>

这里记录的2个节点，均指存储节点（storage node）.
官方文档这里对每个节点均设置了2个硬盘（/dev/sdb, /dev/sdb）.


### storage node {#storage-node}

<https://docs.openstack.org/swift/queens/install/storage-install-ubuntu-debian.html>


#### storage node 可以使用已挂载硬盘的某个新分区 {#storage-node-可以使用已挂载硬盘的某个新分区}

Format the /dev/sdb and /dev/sdc devices as XFS:

```shell
$ mkfs.xfs /dev/sda6
```


#### storage node 中的 MANAGEMENT\_INTERFACE\_IP\_ADDRESS {#storage-node-中的-management-interface-ip-address}

<https://docs.openstack.org/swift/queens/install/storage-install-ubuntu-debian.html#install-and-configure-components>

Replace MANAGEMENT\_INTERFACE\_IP\_ADDRESS with the IP address of the management network on the storage node.
这里的 MANAGEMENT\_INTERFACE\_IP\_ADDRESS 应该理解为 storage node 上的对应与
management network 中的IP，也就是 192.168.100.105 之类的IP


### rings {#rings}

<https://docs.openstack.org/swift/queens/install/initial-rings.html#distribute-ring-configuration-files>

这一步不要忘记了。也就是把 controller node 中的 account.ring.gz, container.ring.gz, and object.ring.gz 复制过去。


### finalize-installation {#finalize-installation}

<https://docs.openstack.org/swift/queens/install/finalize-installation-ubuntu-debian.html>

On the controller node and any other nodes running the proxy service, restart the Object Storage proxy service including its dependencies:

```shell
$ service memcached restart
$ service swift-proxy restart
```

这里运行完成后，必须要用 status 检查一下。当时我就是没有检查（实际上是storage node没有开放相应端口），导致后面一步的 \`swift stat\` 失效。

```shell
$ service memcached status
$ service swift-proxy status
```


### verify {#verify}

<https://docs.openstack.org/swift/queens/install/verify.html>

基本的命令，也可以通过 horizon 进行。

demo-openrc 文件应该之前就有了。

```shell
root@controller:~# cat demo-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@controller:~# . demo-openrc
```


#### 上传文件 {#上传文件}

```shell
root@controller:~# openstack object create container1 360bdoctor.exe
root@controller:~# openstack object list container1
+----------------+
| Name           |
+----------------+
| 360bdoctor.exe |
+----------------+
root@controller:~#
```

-    当上传文件太大时，会出503错误。

    ```shell
    root@controller:~# openstack object create container1 cn_sql_server_2008_r2_enterprise_x86_x64_ia64_dvd_522233.rar -v
    START with options: [u'object', u'create', u'container1', u'cn_sql_server_2008_r2_enterprise_x86_x64_ia64_dvd_522233.rar', u'-v']
    command: object create -> openstackclient.object.v1.object.CreateObject (auth=True)
    Using auth plugin: password
    Service Unavailable (HTTP 503) (Request-ID: tx063c37fe0f004a6fba14c-005c345bf6)
    END return value: 1
    root@controller:~# du -sh cn_sql_server_2008_r2_enterprise_x86_x64_ia64_dvd_522233.rar
    3.3G	cn_sql_server_2008_r2_enterprise_x86_x64_ia64_dvd_522233.rar
    root@controller:~#
    ```

    是不是与配置过程中的 <https://docs.openstack.org/swift/queens/install/initial-rings.html> For simplicity, this guide uses one region and two zones with 2^10 (1024) maximum partitions, 3 replicas of each object, and 1 hour minimum time between moving a partition more than once. 中的
    2^10 (1024) maximum partitions 这个位置相关。如果是这个思路，我们可以尝试往这个方向，改一下。

-    在上传文件时，请不要使用 "./\*\*\*\*" 形式

    因为这样的话，会导致在 horizon 中显示为目录形式，这明显不对。
    而且，当有这样的目录文件时，在 horizon 是删除不了这个文件的（可能是因为，horizon 传过去的就是 "." ，而不是你希望的 "./360bdoctor.exe" 这样的文件）
    所以这时，只能通过命令行\`openstack object delete controller1 ./360bdoctor.exe\`这样的删除了。

    (下面命令请不要运行)

    ```shell
    $ openstack object create container1 ./360bdoctor.exe
    ```

-    不能上传文件夹

    官网的这次配置不能上传文件夹，是不是与配置过程中的某个因素相关

    ```shell
    root@controller:~# ll
    total 3450604
    drwx------  8 root root       4096 Jan  8 15:52 ./
    drwxr-xr-x 23 root root       4096 Oct 22 18:11 ../
    -rw-r--r--  1 root root    1993728 Jan  4 18:58 360bdoctor.exe
    drwx------  2 root root       4096 Oct 29 10:50 .ssh/
    root@controller:~# openstack object create container1 ttt -v
    START with options: [u'object', u'create', u'container1', u'ttt', u'-v']
    command: object create -> openstackclient.object.v1.object.CreateObject (auth=True)
    Using auth plugin: password
    [Errno 21] Is a directory: u'ttt'
    END return value: 1
    root@controller:~#
    ```


## Ref {#ref}

-   <https://docs.openstack.org/swift/queens/install/>
