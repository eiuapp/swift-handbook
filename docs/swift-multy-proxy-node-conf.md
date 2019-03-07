
Openstack存储swift多代理节点安装配置

![swift-multy-proxy-node-architecture](https://res.cloudinary.com/dmtixvmgt/image/upload/v1548240314/swift-multy-proxy-node-architecture_cjcb3m.jpg)

## env

- 系统：Ubuntu Server 16.04×64 
- 存储设置：4T
- 架构部署: 

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| Proxy |  192.168.0.51 | 代理节点, controller node |
| object1      |     192.168.0.127    |     存储节点1(zone1)| 
| object2      |     192.168.0.134    |     存储节点2(zone1)| 
| object3      |     192.168.0.135    |     存储节点3(zone1)| 
| object4      |    192.168.0.180     |     存储节点4(zone1)| 
| object5      |    192.168.0.189     |     存储节点5(zone1)| 

增加代理节点

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| Proxybak  |        192.168.0.141   |    代理节点, 做冗余备份| 

## 注意

- /etc/swift/proxy-swift.conf 中 关于auth的部分,IP 一定要指向 keystone IP。
- 在使用时，OS_AUTH_URL 中体现的 IP 一定要指向 keystone。如：`OS_AUTH_URL=http://controller:5000/v3` 中的controller 的IP要指向 keystone IP。 
- swift服务endpoint的IP要指向 swift proxy 的IP。如：本环境中，指向 192.168.0.51或者192.168.0.141。
- 新增节点的python请一定要能`import memcache`。


## step

### swift 环境

之前已经正常安装好了proxy node与 object1-object5. 具体步骤, 参考前文，这里省略。

下面讲一下，新增 proxybak 节点的部分。

### 增加代理节点从这里开始

#### proxy node

准备一些文件

```shell
root@controller:~# cp  -a /etc/swift/ /tmp/swift/
root@controller:~# cp /etc/hosts /tmp/
root@controller:~# 
```

#### proxybak node

安装依赖
```shell
apt-get install swift swift-proxy python-swiftclient \
  python-keystoneclient python-keystonemiddleware \
  memcached
```

拿文件

`/etc/hosts` 文件请与proxy node保持一致。`ping controller` 保证连接到 controller node。

```shell
root@ubuntu:~# scp -r ubuntu@192.168.0.51:/tmp/swift/ .
root@ubuntu:~# scp -r ubuntu@192.168.0.51:/tmp/hosts .
root@ubuntu:~# tail -10 hosts >> /etc/hosts
root@ubuntu:~# cat /etc/hosts
root@ubuntu:~# ping controller
PING controller (192.168.0.51) 56(84) bytes of data.
64 bytes from controller (192.168.0.51): icmp_seq=1 ttl=64 time=45.1 ms
^C
--- controller ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 45.145/45.145/45.145/0.000 ms
root@ubuntu:~# 

```

准备配置

```shell
root@ubuntu:~# mv swift/ /etc/swift/
root@ubuntu:~# ls /etc/swift/
```

什么也不要修改。如果说你有需求要修改配置，请注意：

- swift.conf配置文件的hash值不要修改，请保持一致。
- 修改proxy-server.conf 中的IP 设置为Proxybak的IP

比如，这里就是把bind_ip设置为 192.168.0.141。

```shell
root@ubuntu:/etc/swift# head proxy-server.conf 
[DEFAULT]
bind_ip = 192.168.0.141 
bind_port = 8080
root@ubuntu:/etc/swift# 
```

开启swift-proxy

```shell
root@ubuntu:/etc/swift# chown -R root:swift /etc/swift
root@ubuntu:/etc/swift# service memcached restart
root@ubuntu:/etc/swift# service swift-proxy restart
root@ubuntu:/etc/swift# service memcached status 
root@ubuntu:/etc/swift# service swift-proxy status 
```

开启 swift-proxy 后，会在本地启动 8080 端口

```shell
root@ubuntu:/etc/swift# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN      7517/memcached  
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      7818/python     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      3415/sshd       
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      3572/0          
tcp        0      0 127.0.0.1:6011          0.0.0.0:*               LISTEN      6366/1          
tcp6       0      0 :::22                   :::*                    LISTEN      3415/sshd       
tcp6       0      0 ::1:6010                :::*                    LISTEN      3572/0          
tcp6       0      0 ::1:6011                :::*                    LISTEN      6366/1          
root@ubuntu:/etc/swift# 
```

在 proxybak node 上访问 swift 服务。

```shell
root@ubuntu:/etc/swift# cat demo-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@ubuntu:/etc/swift# . demo-openrc 
root@ubuntu:/etc/swift# swift list
container1
container2
root@ubuntu:/etc/swift# 
```

结果正常。说明proxybak node 可用。
增加代理节点成功，操作结束。


## ref

- https://docs.openstack.org/swift/latest/install/controller-install-ubuntu.html#install-and-configure-components
