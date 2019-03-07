

让 openstack dev 2 环境中的swift 通过 keystone 认证

## env

- 系统：Ubuntu Server 16.04×64 
- 存储设置：4T
- 架构部署: 

主机名                 IP                    作用
Proxy            192.168.0.51         代理节点, controller node
object1          192.168.0.127        存储节点1(zone1)
object2          192.168.0.134        存储节点2(zone1)
object3          192.168.0.135        存储节点3(zone1)
object4          192.168.0.180        存储节点4(zone1)
object5          192.168.0.189        存储节点5(zone1)

增加代理节点
Proxybak         192.168.0.141      代理节点, 做冗余备份

## step

### 配置
只需要把下面的文本放至 `/etc/keystone/default_catalog.templates` 末尾


```text
catalog.RegionOne.object_store.name = Swift Service
catalog.RegionOne.object_store.publicURL = http://swiftproxy:8080/v1/AUTH_$(tenant_id)s
catalog.RegionOne.object_store.adminURL = http://swiftproxy:8080/
catalog.RegionOne.object_store.internalURL = http://swiftproxy:8080/v1/AUTH_$(tenant_id)s
```

### 重启 keystone 

```shell
service apache2 restart
``` 

### 检查
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
root@controller:~# swift list
container1
container2
root@controller:~# 
``` 
## ref
- https://docs.openstack.org/swift/latest/overview_auth.html