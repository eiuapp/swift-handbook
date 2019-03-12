在 192.168.10.12 上安装 proxy node 

## env

因为 192.168.10.13 中的 用户信息获取时总是有问题。所以，组员决定不折腾（更折腾）在另一台机器上重新安装 proxy , keystone, controller 组件。

proxy 搭建 对应的controller，keystone 均为 192.168.10.12  

## step 

```
root@ubuntu:~# sudo apt-get install swift swift-proxy python-swiftclient \
>   python-keystoneclient python-keystonemiddleware \
>   memcached -y
```

发现没有 /etc/swift/ 文件夹，那么把 192.168.10.13 上的 /etc/swift/ 直接复制过来。

```
root@ubuntu:~# ls /etc/swift
ls: cannot access '/etc/swift': No such file or directory
root@ubuntu:~# cp -a /home/administrator/swift/ /etc/swift/
root@ubuntu:~# cd /etc/swift/
root@ubuntu:/etc/swift# head proxy-server.conf 
[DEFAULT]
# bind_ip = 0.0.0.0
bind_port = 8080
# bind_timeout = 30
# backlog = 4096
swift_dir = /etc/swift
user = swift

# Enables exposing configuration settings via HTTP GET /info.
# expose_info = true
root@ubuntu:/etc/swift# chown -R root:swift /etc/swift
```

修改 /etc/hosts 文件

```
root@ubuntu:~# vi /etc/hosts
root@ubuntu:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu
#192.168.10.12	controller
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


# controller
192.168.10.13      controller

# swiftproxy 
192.168.10.12      swiftproxy
root@ubuntu:~#
```

确保有 memcache 包

```
root@ubuntu:/etc/swift# python
Python 2.7.12 (default, Nov 12 2018, 14:36:49) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import memcache
>>> 
root@ubuntu:/etc/swift# 
```

```
root@ubuntu:/etc/swift# service memcached restart
root@ubuntu:/etc/swift# service swift-proxy restart
root@ubuntu:/etc/swift# service memcached status 
root@ubuntu:/etc/swift# service memcached status  
root@ubuntu:/etc/swift# service swift-proxy status 
root@ubuntu:/etc/swift# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      7904/sshd       
tcp        0      0 127.0.0.1:7001          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      15627/0         
tcp        0      0 127.0.0.1:4001          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      9901/beam.smp   
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      8792/mysqld     
tcp        0      0 0.0.0.0:11211           0.0.0.0:*               LISTEN      16688/memcached 
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 127.0.0.1:2380          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      16714/python    
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      9788/epmd       
tcp6       0      0 :::22                   :::*                    LISTEN      7904/sshd       
tcp6       0      0 ::1:6010                :::*                    LISTEN      15627/0         
tcp6       0      0 :::5000                 :::*                    LISTEN      13217/apache2   
tcp6       0      0 :::5672                 :::*                    LISTEN      9901/beam.smp   
tcp6       0      0 :::80                   :::*                    LISTEN      13217/apache2   
tcp6       0      0 :::4369                 :::*                    LISTEN      9788/epmd       
root@ubuntu:/etc/swift# 
```


```
root@ubuntu:~# cd
root@ubuntu:~# ls
demo-openrc
root@ubuntu:~# vi demo-openrc 
root@ubuntu:~# . demo-openrc 
root@ubuntu:~# swift list
ab
abc
abcd
container1
test
root@ubuntu:~# 
```

这里还是 192.168.10.13 上的数据

查一下 endpoint 

```
root@ubuntu:~# cat demo-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=413eaac0d4cff66fde56
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

root@ubuntu:~# 
root@ubuntu:~# openstack endpoint list
+----------------------------------|-----------|--------------|--------------|---------|-----------|--------------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                              |
+----------------------------------|-----------|--------------|--------------|---------|-----------|--------------------------------------------------+
| 3736d384b23f45a8a21e6f5e5888d759 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.10.13:5000/v3/                    |
| 3fc42de3790b4c5f8d635a9721ed23a6 | RegionOne | swift        | object-store | True    | public    | http://192.168.10.13:8080/v1/AUTH_%(project_id)s |
| 88634b7b1e6c4b0989f079f00adbffb8 | RegionOne | keystone     | identity     | True    | internal  | http://192.168.10.13:5000/v3/                    |
| 8b4faa17e4404ef4aa709c0e78636742 | RegionOne | swift        | object-store | True    | internal  | http://192.168.10.13:8080/v1/AUTH_%(project_id)s |
| bf0706eea0e04412ae4588ec9dd067a0 | RegionOne | keystone     | identity     | True    | public    | http://192.168.10.13:5000/v3/                    |
| bfae1eb954c84364803023604999ef93 | RegionOne | swift        | object-store | True    | admin     | http://192.168.10.13:8080/v1/                    |
+----------------------------------|-----------|--------------|--------------|---------|-----------|--------------------------------------------------+
root@ubuntu:~# 
```

这里 肯定要修改一下 endpoint, 让 swift 的服务 url 指向 192.168.10.12

修改 /etc/hosts

```
root@ubuntu:~# vi /etc/hosts
root@ubuntu:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu
#192.168.10.12	controller
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


# controller
192.168.10.12      controller

# swiftproxy 
192.168.10.12      swiftproxy
root@ubuntu:~#
```

重启服务，发现 openstack 认证不了。要去找 admin 用户。

```
root@ubuntu:~# service swift-proxy restart
root@ubuntu:~# service memcached restart
root@ubuntu:~# openstack endpoint list
The request you have made requires authentication. (HTTP 401) (Request-ID: req-afb2463d-ba10-455b-a3af-5ebd0b1a1bb4)
root@ubuntu:~# 
```

```
root@ubuntu:~# vi admin-openrc
root@ubuntu:~# . admin-openrc 
root@ubuntu:~# openstack endpoint list
+----------------------------------|-----------|--------------|--------------|---------|-----------|-------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                           |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-------------------------------+
| 3424fdd10aa047418502f4fe92f1f10c | RegionOne | keystone     | identity     | True    | public    | http://192.168.10.12:5000/v3/ |
| 698a2a4b64b74e5ea289c21296e36303 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.10.12:5000/v3/ |
| d66b0bd887d141c49a166c2426633cb8 | RegionOne | keystone     | identity     | True    | internal  | http://192.168.10.12:5000/v3/ |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-------------------------------+
root@ubuntu:~# openstack user list
+----------------------------------|-------+
| ID                               | Name  |
+----------------------------------|-------+
| 590061f35488419e8aa33fd9cfc9ce3f | demo  |
| e73b44116b8a4a0aacb9ad6b4f08d7b8 | admin |
+----------------------------------|-------+
```

创建 domain 

```
root@ubuntu:~# openstack user create --domain default --password-prompt swift 
User Password:
Repeat User Password:
+---------------------|----------------------------------+
| Field               | Value                            |
+---------------------|----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | d1eecb70a7314472b386e8eb2d05aed8 |
| name                | swift                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------|----------------------------------+
root@ubuntu:~# 
```

swift 对应 密码是 `5df7764659a27b695e85`

创建 service , endpoint 


```
root@ubuntu:~# openstack role add --project service --user swift admin
root@ubuntu:~# openstack service create --name swift \
>   --description "OpenStack Object Storage" object-store
+-------------|----------------------------------+
| Field       | Value                            |
+-------------|----------------------------------+
| description | OpenStack Object Storage         |
| enabled     | True                             |
| id          | 25dfd6435b4245a9991f6a0e0c5f609c |
| name        | swift                            |
| type        | object-store                     |
+-------------|----------------------------------+
root@ubuntu:~# openstack endpoint create --region RegionOne \
>   object-store public http://controller:8080/v1/AUTH_%\(project_id\)s
+--------------|-----------------------------------------------+
| Field        | Value                                         |
+--------------|-----------------------------------------------+
| enabled      | True                                          |
| id           | 4327c259c9a843e5b0b9dff1140a1e72              |
| interface    | public                                        |
| region       | RegionOne                                     |
| region_id    | RegionOne                                     |
| service_id   | 25dfd6435b4245a9991f6a0e0c5f609c              |
| service_name | swift                                         |
| service_type | object-store                                  |
| url          | http://controller:8080/v1/AUTH_%(project_id)s |
+--------------|-----------------------------------------------+
root@ubuntu:~# openstack endpoint create --region RegionOne \
>   object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
+--------------|-----------------------------------------------+
| Field        | Value                                         |
+--------------|-----------------------------------------------+
| enabled      | True                                          |
| id           | b87a66d75ea242ef97c1969c0deaff57              |
| interface    | internal                                      |
| region       | RegionOne                                     |
| region_id    | RegionOne                                     |
| service_id   | 25dfd6435b4245a9991f6a0e0c5f609c              |
| service_name | swift                                         |
| service_type | object-store                                  |
| url          | http://controller:8080/v1/AUTH_%(project_id)s |
+--------------|-----------------------------------------------+
root@ubuntu:~# openstack endpoint create --region RegionOne \
>   object-store admin http://controller:8080/v1
+--------------|----------------------------------+
| Field        | Value                            |
+--------------|----------------------------------+
| enabled      | True                             |
| id           | 8eba4b8e86654a999c1cd8fe27f94735 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25dfd6435b4245a9991f6a0e0c5f609c |
| service_name | swift                            |
| service_type | object-store                     |
| url          | http://controller:8080/v1        |
+--------------|----------------------------------+
root@ubuntu:~# openstack endpoint  list
+----------------------------------|-----------|--------------|--------------|---------|-----------|-----------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                           |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-----------------------------------------------+
| 3424fdd10aa047418502f4fe92f1f10c | RegionOne | keystone     | identity     | True    | public    | http://192.168.10.12:5000/v3/                 |
| 4327c259c9a843e5b0b9dff1140a1e72 | RegionOne | swift        | object-store | True    | public    | http://controller:8080/v1/AUTH_%(project_id)s |
| 698a2a4b64b74e5ea289c21296e36303 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.10.12:5000/v3/                 |
| 8eba4b8e86654a999c1cd8fe27f94735 | RegionOne | swift        | object-store | True    | admin     | http://controller:8080/v1                     |
| b87a66d75ea242ef97c1969c0deaff57 | RegionOne | swift        | object-store | True    | internal  | http://controller:8080/v1/AUTH_%(project_id)s |
| d66b0bd887d141c49a166c2426633cb8 | RegionOne | keystone     | identity     | True    | internal  | http://192.168.10.12:5000/v3/                 |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-----------------------------------------------+
```

重启服务

```
root@ubuntu:~# chown -R root:swift /etc/swift
root@ubuntu:~# service swift-proxy restart
root@ubuntu:~# service memcached restart
root@ubuntu:~#  netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      7904/sshd       
tcp        0      0 127.0.0.1:7001          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      15627/0         
tcp        0      0 127.0.0.1:4001          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      9901/beam.smp   
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      8792/mysqld     
tcp        0      0 0.0.0.0:11211           0.0.0.0:*               LISTEN      17294/memcached 
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 127.0.0.1:2380          0.0.0.0:*               LISTEN      11437/etcd      
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      17265/python    
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      9788/epmd       
tcp6       0      0 :::22                   :::*                    LISTEN      7904/sshd       
tcp6       0      0 ::1:6010                :::*                    LISTEN      15627/0         
tcp6       0      0 :::5000                 :::*                    LISTEN      13217/apache2   
tcp6       0      0 :::5672                 :::*                    LISTEN      9901/beam.smp   
tcp6       0      0 :::80                   :::*                    LISTEN      13217/apache2   
tcp6       0      0 :::4369                 :::*                    LISTEN      9788/epmd       
root@ubuntu:~# . admin-openrc 
root@ubuntu:~# swift list
```

用原来 192.168.10.13 的 demo 用户 openrc 文件，已经不可用。


```
root@ubuntu:~# . demo-openrc 
root@ubuntu:~# swift list
Unauthorized. Check username/id, password, tenant name/id and user/tenant domain name/id.
root@ubuntu:~# 
```

创建新的 demo 用户openrc 文件。

```
root@ubuntu:~# cp demo-openrc demo-openrc.12
root@ubuntu:~# vi demo-openrc.12  
root@ubuntu:~# cat demo-openrc.12
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@ubuntu:~# . demo-openrc.12 
root@ubuntu:~# swift list
root@ubuntu:~# 
```

好了，现在可以使用了。

这样，全新的一个 proxy node 节点就搭建好了。



## ref
- https://docs.openstack.org/swift/queens/install/controller-install-ubuntu.html
- https://eiuapp.github.io/swift-handbook/docs/swift-multy-proxy-node-conf.html
