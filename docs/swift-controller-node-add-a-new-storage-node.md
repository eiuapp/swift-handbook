
controller node add a new storage node 


## 注意：

当controller node 做了 rebalance 后，希望 oss 保持最新，则必须：

1. 向所有的storage node 分发最新的 /etc/swift/*.ring.gz 文件
2. 在所有的 storage node 重启 swift-init 

## step

### 修改hosts

```shell
tail -1 hosts >> /etc/hosts
```

### 删除老节点

```shell
## remove 
swift-ring-builder object.builder
swift-ring-builder object.builder remove --id=1   
swift-ring-builder container.builder
swift-ring-builder container.builder  remove --id=1
swift-ring-builder account.builder 
swift-ring-builder account.builder remove --id=1   

## check
swift-ring-builder object.builder
swift-ring-builder container.builder
swift-ring-builder account.builder 

## add 
swift-ring-builder object.builder add   --region 1 --zone 1 --ip 192.168.0.127 --port 6200 --device sdb --weight 100
swift-ring-builder container.builder add   --region 1 --zone 1 --ip 192.168.0.127 --port 6201 --device sdb --weight 100
swift-ring-builder account.builder   add --region 1 --zone 1 --ip 192.168.0.127 --port 6202   --device sdb --weight 100 
swift-ring-builder object.builder add   --region 1 --zone 1 --ip 192.168.0.180 --port 6200 --device sdb --weight 100
swift-ring-builder container.builder add   --region 1 --zone 1 --ip 192.168.0.180 --port 6201 --device sdb --weight 100
swift-ring-builder account.builder   add --region 1 --zone 1 --ip 192.168.0.180 --port 6202   --device sdb --weight 100 

## check
swift-ring-builder object.builder
swift-ring-builder container.builder
swift-ring-builder account.builder 

## rebalance
swift-ring-builder object.builder rebalance
swift-ring-builder account.builder rebalance
swift-ring-builder container.builder rebalance

## check
swift-ring-builder object.builder
swift-ring-builder container.builder
swift-ring-builder account.builder 
``` 

### 传*.ring.gz文件给其它node

```shell
cp /etc/swift/*.ring.gz /tmp/storage-node/swift/
``` 

### storage node get the ring.gz 

```shell 
scp ubuntu@192.168.0.51:/tmp/storage-node-files/storage-node/ring.gz/  /tmp/storage-node/
```

```shell
cp /tmp/storage-node/ring.gz/* /etc/swift/
chown -R root:swift /etc/swift
``` 

### On the controller node 
```shell
service memcached restart
service swift-proxy restart
service memcached status
service swift-proxy status
``` 

### On the storage nodes, start the Object Storage services:

```shell
service rsync stop
service rsync start
service rsync status
swift-init all restart
```


 