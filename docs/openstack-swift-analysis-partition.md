
ring 生成 partition 分析

## env

- 系统：Ubuntu Server 16.04×64 
- 存储设置：4T
- python-swiftclient 3.5.0
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

#### 网上示例

通过 https://ask.openstack.org/en/question/6766/what-is-a-swift-partition/ 知道：

SEC/MY_account/cats/kitten.jpgRET => 4f3146f77835ff33a6baba807f7f5c10 => 4f31

- SEC/MY_account/cats/kitten.jpgRET => 4f3146f77835ff33a6baba807f7f5c10 这一步，放到[网上md5加密](https://md5jiami.51240.com/)就能得到了。
- 4f3146f77835ff33a6baba807f7f5c10 => 4f31
- 4f31 => 20273 这一步，放到[网上16进制转10进制](http://tool.oschina.net/hexconvert)就能得到了。

那么我们这个版本的是如何生成 partition 的呢？

#### 现象

从下面的现象，我们知道，当传入 

account= "AUTH_7d6eaa90d74a4f239963933c3a744df3"
container = "test4"
obj = "object-a.txt"

最后得到的 partition 值是 119

怎么得到下面的 119 呢？

#### 本质

直接看源码。

##### 源码位置

在 python 安装包的路径下。我这里是 

```shell
root@controller:/usr/lib/python2.7/dist-packages/swift# ls
account  cli  common  container  __init__.py  __init__.pyc  locale  obj  proxy
root@controller:/usr/lib/python2.7/dist-packages/swift# 
```

如果你当前没有安装swift的环境，也没有关系，去[github下载](https://github.com/openstack/swift)。
这里要注意版本。不同版本，源代码当然不同啦, 找与你对应的版本哟! 我的swift是queen版本时安装的。
下载后，有一个，进入，里有还有一个swift文件夹，此文件夹中的内容，与上面我这里的是一致的。

各文件夹是什么作用，这里先不介绍了。

##### get_part

在 `./common/ring/ring.py` 的 333 行，找到 get_part 函数。这个就是我想要的。

```python
    def get_part(self, account, container=None, obj=None):
        """
        Get the partition for an account/container/object.

        :param account: account name
        :param container: container name
        :param obj: object name
        :returns: the partition number
        """
        key = hash_path(account, container, obj, raw_digest=True)
        if time() > self._rtime:
            self._reload()
        part = struct.unpack_from('>I', key)[0] >> self._part_shift
        return part
```

我们分析一下。里面主要就是 hash_path 和 struct.unpack_from 函数

###### hash_path

找到源码 `common/utils.py` 的 2265 行，找到 hash_path 函数

```python
def hash_path(account, container=None, object=None, raw_digest=False):
    """
    Get the canonical hash for an account/container/object

    :param account: Account
    :param container: Container
    :param object: Object
    :param raw_digest: If True, return the raw version rather than a hex digest
    :returns: hash string
    """
    if object and not container:
        raise ValueError('container is required if object is provided')
    paths = [account]
    if container:
        paths.append(container)
    if object:
        paths.append(object)
    if raw_digest:
        return md5(HASH_PATH_PREFIX + '/' + '/'.join(paths)
                   + HASH_PATH_SUFFIX).digest()
    else:
        return md5(HASH_PATH_PREFIX + '/' + '/'.join(paths)
                   + HASH_PATH_SUFFIX).hexdigest()
```


看到了么？我们这里是执行 

```
md5(HASH_PATH_PREFIX + '/' + '/'.join(paths)
                   + HASH_PATH_SUFFIX).digest()
```

注意到这里有 

- HASH_PATH_PREFIX = "openstack"
- HASH_PATH_SUFFIX = "openstack"

来自于配置文件`/etc/swift/swift.conf`

```shell
root@controller:/home/ubuntu# grep swift_hash_path_suffix -rn /etc/swift/swift.conf
9:swift_hash_path_suffix = openstack
root@controller:/home/ubuntu# grep swift_hash_path_prefix -rn /etc/swift/swift.conf
10:swift_hash_path_prefix = openstack
root@controller:/home/ubuntu# 
```
####### md5

https://docs.python.org/2/library/md5.html

###### struct.unpack_from

struct.unpack_from 的 作用，请看[这里](/jwl/python-struct-module/) 或者 [这里](/post/python-struct-module/)

简单说就是： 将字节流转换成python数据类型。Unpack the buffer according to the given format.

我们这里是 `struct.unpack_from('>I', key)[0]` 

`'>I'`, 表示转换成python的 unsigned int。

#### 本质的一个示例

```shell
ubuntu@controller:~$ python
Python 2.7.12 (default, Dec  4 2017, 14:50:18) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import swift
>>> from swift.common.utils import hash_path, validate_configuration
>>> from hashlib import md5, sha1
>>> import struct
>>> HASH_PATH_PREFIX = "openstack"
>>> HASH_PATH_SUFFIX = "openstack"
>>> account= "AUTH_7d6eaa90d74a4f239963933c3a744df3"
>>> container = "test4"
>>> obj = "object-a.txt"
>>> paths = [account, container, obj]
>>> '/'.join(paths)
'AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a.txt'
>>> HASH_PATH_PREFIX + '/' + '/'.join(paths) + HASH_PATH_SUFFIX
'openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a.txtopenstack'
>>> md5(HASH_PATH_PREFIX + '/' + '/'.join(paths) + HASH_PATH_SUFFIX)
<md5 HASH object @ 0x7f8632c45670>
>>> md5(HASH_PATH_PREFIX + '/' + '/'.join(paths) + HASH_PATH_SUFFIX).digest()
'\x1d\xf3\xd1_:\x9a+x\x8e75v\xbc\xf9v\x11'
>>> key = hash_path(account, container, obj, raw_digest=True)
>>> key
'\x1d\xf3\xd1_:\x9a+x\x8e75v\xbc\xf9v\x11'
>>> print(struct.unpack_from('>I', key))
(502518111,)
>>> print(struct.unpack_from('>I', key)[0])
502518111
>>> 502518111 >> 22
119
>>> part = struct.unpack_from('>I', key)[0] >> 22
>>> part
119
>>> exit()
```

好了，现在基本上知道，119，就是这么来的了。

#### 调试源代码

但是，如果你看是想亲眼见一下，119，怎么办？调试源代码吧。

因为我之前已经上传了 object = "object-a.txt" 。不能同名的。所以我现在object = "object-a-3.txt"

##### 修改源代码

我这里只是示例。

- `./common/ring/ring.py` 的 333 行

```python
    def get_part(self, account, container=None, obj=None):
        """
        Get the partition for an account/container/object.

        :param account: account name
        :param container: container name
        :param obj: object name
        :returns: the partition number
        """
        key = hash_path(account, container, obj, raw_digest=True)
        print("--------------------------------------------------, my debug --------------------")
        print("=========keys:", key)
        if time() > self._rtime:
            self._reload()
        part = struct.unpack_from('>I', key)[0] >> self._part_shift
        print("11111111partition:", part)
        return part
```
- `common/utils.py` 的 2265 行

```python
def hash_path(account, container=None, object=None, raw_digest=False):
    """
    Get the canonical hash for an account/container/object

    :param account: Account
    :param container: Container
    :param object: Object
    :param raw_digest: If True, return the raw version rather than a hex digest
    :returns: hash string
    """
    if object and not container:
        raise ValueError('container is required if object is provided')
    paths = [account]
    if container:
        paths.append(container)
    if object:
        paths.append(object)
    if raw_digest:
        print("--------------------------------------------------, my debug --------------------")
        print(HASH_PATH_PREFIX)
        print(HASH_PATH_PREFIX + '/' + '/'.join(paths) + HASH_PATH_SUFFIX)
        print(md5(HASH_PATH_PREFIX + '/' + '/'.join(paths) + HASH_PATH_SUFFIX).digest())

        return md5(HASH_PATH_PREFIX + '/' + '/'.join(paths)
                   + HASH_PATH_SUFFIX).digest()
    else:
        return md5(HASH_PATH_PREFIX + '/' + '/'.join(paths)
                   + HASH_PATH_SUFFIX).hexdigest()
```

##### 重启swift-proxy服务

```shell
root@controller:/home/ubuntu# service swift-proxy restart
root@controller:/home/ubuntu# service swift-proxy status
```

##### 看日志

```shell
root@controller:/home/ubuntu# tail -f /var/log/syslog
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ??70L??023G (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('=========keys:', '\xdf\xcf\x8a\xff\x9b7\xc1H0L\xf4A\xb2\x13\xc1\x87') (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('11111111partition:', 895) (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: - - 30/Jan/2019/08/07/44 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 204 - Swift - - - - tx5d49fd7b64cb4e0c89e04-005c515b50 - 0.0047 RL - 1548835664.549806118 1548835664.554487944 -
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ??70L??023G (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('=========keys:', '\xdf\xcf\x8a\xff\x9b7\xc1H0L\xf4A\xb2\x13\xc1\x87') (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('11111111partition:', 895) (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ?]?14Ν?#023#032#032#001#027??txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('=========keys:', '\xbc]\xec\x0c\xcf]\x8a\xb6\x13\x1a\x96\x1a\x01\x17\xf6\xd1') (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('11111111partition:', 753) (txn: tx5d49fd7b64cb4e0c89e04-005c515b50)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: ?]?14Ν?#023#032#032#001#027??txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('=========keys:', '\xbc]\xec\x0c\xcf]\x8a\xb6\x13\x1a\x96\x1a\x01\x17\xf6\xd1') (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('11111111partition:', 753) (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a-3.txtopenstack (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: >惘&#013??#026%?? (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('=========keys:', '>\xe6\x83\x98&\x0b\xaa\xe1\xe0\xa1\x16%\xd9\xf6\x82\xbf') (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: STDOUT: ('11111111partition:', 251) (txn: tx5d49fd7b64cb4e0c89e04-005c515b50) (client_ip: 127.0.0.1)
Jan 30 16:07:44 controller proxy-server: 127.0.0.1 127.0.0.1 30/Jan/2019/08/07/44 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a-3.txt HTTP/1.0 404 - python-swiftclient-3.5.0 gAAAAABcUVjCZNjJ... - - - tx5d49fd7b64cb4e0c89e04-005c515b50 - 0.0212 - - 1548835664.548610926 1548835664.569818020 0
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4openstack (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ?]?14Ν?#023#032#032#001#027??txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xbc]\xec\x0c\xcf]\x8a\xb6\x13\x1a\x96\x1a\x01\x17\xf6\xd1') (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 753) (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a-3.txtopenstack (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: >惘&#013??#026%?? (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '>\xe6\x83\x98&\x0b\xaa\xe1\xe0\xa1\x16%\xd9\xf6\x82\xbf') (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 251) (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: 127.0.0.1 127.0.0.1 30/Jan/2019/08/07/45 PUT /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a-3.txt HTTP/1.0 201 - python-swiftclient-3.5.0 gAAAAABcUVjCZNjJ... 18431 - - txacc37f453a6441499423e-005c515b51 - 0.0709 - - 1548835665.213526964 1548835665.284423113 0
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3openstack (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ??70L??023G (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xdf\xcf\x8a\xff\x9b7\xc1H0L\xf4A\xb2\x13\xc1\x87') (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 895) (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4openstack (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ?]?14Ν?#023#032#032#001#027??txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xbc]\xec\x0c\xcf]\x8a\xb6\x13\x1a\x96\x1a\x01\x17\xf6\xd1') (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 753) (txn: tx06a68fa051174db18c0aa-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3openstack (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ??70L??023G (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xdf\xcf\x8a\xff\x9b7\xc1H0L\xf4A\xb2\x13\xc1\x87') (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 895) (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4openstack (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ?]?14Ν?#023#032#032#001#027??txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xbc]\xec\x0c\xcf]\x8a\xb6\x13\x1a\x96\x1a\x01\x17\xf6\xd1') (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 753) (txn: tx8a9fb911df1d48af97b6f-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: 127.0.0.1 127.0.0.1 30/Jan/2019/08/07/45 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/%3Fformat%3Djson HTTP/1.0 200 - python-swiftclient-3.5.0 gAAAAABcUVjCZNjJ... - 103 - tx06a68fa051174db18c0aa-005c515b51 - 0.0584 - - 1548835665.329989910 1548835665.388351917 0
Jan 30 16:07:45 controller proxy-server: 127.0.0.1 127.0.0.1 30/Jan/2019/08/07/45 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4%3Fdelimiter%3D%252F%26limit%3D1001%26format%3Djson HTTP/1.0 200 - python-swiftclient-3.5.0 gAAAAABcUVjCZNjJ... - 1017 - tx8a9fb911df1d48af97b6f-005c515b51 - 0.0464 - - 1548835665.346412897 1548835665.392823935 0
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3openstack (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ??70L??023G (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xdf\xcf\x8a\xff\x9b7\xc1H0L\xf4A\xb2\x13\xc1\x87') (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 895) (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4openstack (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ?]?14Ν?#023#032#032#001#027??txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '\xbc]\xec\x0c\xcf]\x8a\xb6\x13\x1a\x96\x1a\x01\x17\xf6\xd1') (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 753) (txn: tx10ed616c142448debc5d4-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: 127.0.0.1 127.0.0.1 30/Jan/2019/08/07/45 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4%3Fmarker%3Dobject-server.txt%26delimiter%3D%252F%26limit%3D1001%26format%3Djson HTTP/1.0 200 - python-swiftclient-3.5.0 gAAAAABcUVjCZNjJ... - 2 - tx10ed616c142448debc5d4-005c515b51 - 0.0069 - - 1548835665.394655943 1548835665.401551962 0
```

找到了

```text
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: openstack/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a-3.txtopenstack (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: >惘&#013??#026%?? (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: --------------------------------------------------, my debug -------------------- (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('=========keys:', '>\xe6\x83\x98&\x0b\xaa\xe1\xe0\xa1\x16%\xd9\xf6\x82\xbf') (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: STDOUT: ('11111111partition:', 251) (txn: txacc37f453a6441499423e-005c515b51) (client_ip: 127.0.0.1)
Jan 30 16:07:45 controller proxy-server: 127.0.0.1 127.0.0.1 30/Jan/2019/08/07/45 PUT /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a-3.txt HTTP/1.0 201 - python-swiftclient-3.5.0 gAAAAABcUVjCZNjJ... 18431 - - txacc37f453a6441499423e-005c515b51 - 0.0709 - - 1548835665.213526964 1548835665.284423113 0
```

这下终于放心了。

几个想看到的点，都看到了。最后得到的 partition 是 251。哈哈！~

#### 现象结合本质

openstackAUTH_7d6eaa90d74a4f239963933c3a744df3/test4/object-a.txtopenstack

Compute the MD5 hash of that string to get 942354320ddd6c54d9a4c052ef18bc25

then take the first 16 bits to get 9423, or in decimal, *****.

## ref

- https://ask.openstack.org/en/question/6766/what-is-a-swift-partition/
- https://docs.openstack.org/swift/queens/overview_ring.html