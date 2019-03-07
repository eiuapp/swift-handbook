
Q：swift controller node rebalance后 部分 storage node 没有得到均衡数据，怎么办？

A：修改 对应的 storage node 的 /etc/rsyncd.conf，把 `address = MANAGEMENT_INTERFACE_IP_ADDRESS` 替换为 `address = 0.0.0.0` , 并重启rsync 服务

```shell
service rsync restart
```

