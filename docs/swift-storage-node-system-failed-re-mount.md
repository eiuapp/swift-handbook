
## 已使用 storage node 的主板坏了，重新换新主板，并挂载。

### 开机

### 修改 hostname

```shell
hostnamectl set-hostname swift0134
```

### 修改 hosts 

```shell
vi /etc/hosts
```

如果是新storage node, 开机后查看到 df -h 已有挂载新硬盘 则运行下面：

```shell
umount `df -h | grep "sda1" | awk '{print $NF}'`
```

总之，最后的结果是要保持 fdisk -l 有 /dev/sda1 , 而 df -h 中无 /dev/sda1 的状态。

### 打开 https://docs.openstack.org/swift/queens/install/storage-install-ubuntu-debian.html 按照下面进行：

```shell
sudo su 
newstoragenodeip=192.168.0.134
```

安装依赖，并挂载磁盘

```shell
apt-get install xfsprogs rsync -y
mkdir -p /srv/node/sdb
echo "/dev/sda1 /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2" >> /etc/fstab
mkfs.xfs -f /dev/sda1
mount /srv/node/sdb
apt-get install -y swift swift-account swift-container swift-object
```

复制controller node 上准备好的文件。

```shell
## copy files 
controller=192.168.0.51
scp -r ubuntu@$controller:/tmp/storage-node/ /tmp/
cp /tmp/storage-node/hosts /etc/hosts
cp /tmp/storage-node/rsync/rsync /etc/default/rsync
cp /tmp/storage-node/rsync/rsyncd.conf /etc/rsyncd.conf
cp /tmp/storage-node/swift/* /etc/swift/
grep "MANAGEMENT_INTERFACE_IP_ADDRESS" -rl /etc/rsyncd.conf | xargs sed -i "s/MANAGEMENT_INTERFACE_IP_ADDRESS/$newstoragenodeip/g"

## chown 
chown -R swift:swift /srv/node
mkdir -p /var/cache/swift
chown -R root:swift /var/cache/swift
chmod -R 775 /var/cache/swift
```


### 上传文件 upload
http://192.168.100.50/horizon/project/containers/container/test3

### check 文件大小
```shell
du -sh /srv/node/sd*/
``` 


