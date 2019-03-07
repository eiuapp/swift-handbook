
Openstack存储swift多代理节点效果展示

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

直接看图

[swift-multy-proxy-node-illustration](https://res.cloudinary.com/dmtixvmgt/image/upload/v1548235432/swift-multy-proxy-node-illustration_vpe9uw.png)

### 在 141 上

先使用 192.168.0.51 作为 proxy，访问正常。

把 51 关闭，访问不正常。

### 在 51 上

使用 141 作为 proxy, 访问正常。

### 在 141 上

使用 141 作为 proxy, 访问正常。

### 结论

51或者141 中，只有要一个 proxy 节点存在，整个 swift 服务都可以对外提供服务。耶耶耶！~

