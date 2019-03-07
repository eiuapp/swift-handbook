
swift使用ResellerAdmin角色来设置account的quota

## env

- 系统：Ubuntu Server 16.04×64 
- 存储设置：4T
- 架构部署: 

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| keystone |  192.168.100.50 | keystone node |
| Proxy |  192.168.100.50/192.168.0.50 | 代理节点  |
| object1      |     192.168.100.105    |     存储节点1(zone1)| 
| object2      |      192.168.100.106   |     存储节点2(zone1)| 
| object3      |      192.168.100.107    |     存储节点3(zone1)| 
| object4      |     192.168.100.107     |     存储节点4(zone1)|  

## 注意

无

## step

### 配置 proxy-swift.conf 

要把 swift proxy 节点的 reseller_admin_role = ResellerAdmin 打开注释，重启服务

```shell
root@controller:~# grep ResellerAdmin -rn /etc/swift/proxy-server.conf  
417:reseller_admin_role = ResellerAdmin
root@controller:~# service swift-proxy restart
root@controller:~# 
```

### 创建新环境

创建新的project, user, role 来实验

准备语句：

```
openstack project create --description "testTenant" testreseller --domain default    
openstack user create --project testreseller --password openstack testreseller 
openstack role create reseller
openstack role add --user testreseller --project testreseller reseller
openstack role add --user testreseller --project testreseller ResellerAdmin
```

开始吧

```shell
root@controller:~# openstack project create   --description "testTenant" testreseller \
>   --domain default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | testTenant                       |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 3320a273a7094d789f99756b0e9d0f4f |
| is_domain   | False                            |
| name        | testreseller                     |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
root@controller:~# openstack project list
+----------------------------------+--------------+
| ID                               | Name         |
+----------------------------------+--------------+
| 3320a273a7094d789f99756b0e9d0f4f | testreseller |
| 7d6eaa90d74a4f239963933c3a744df3 | demo         |
| a640c74e595c44c4902d1c5ebc3afa8a | service      |
| f04ec0abf3d1460dad82608bb03af589 | admin        |
+----------------------------------+--------------+ 
root@controller:~# openstack user create --project testreseller --password openstack testreseller
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| default_project_id  | 3320a273a7094d789f99756b0e9d0f4f |
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c83158ab5c3348ab9d148673c23903be |
| name                | testreseller                     |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+ 
root@controller:~# openstack role create reseller
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 09d4fdc0e9544f509c599cd62ab40e6c |
| name      | reseller                         |
+-----------+----------------------------------+ 
root@controller:~# openstack role add --user testreseller --project testreseller reseller
root@controller:~# 
```

创建 ResellerAdmin ，把新用户 testreseller 赋予这个role

```shell
root@controller:~# openstack role create ResellerAdmin
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 1671364ac7a945e5ae164e100de2c366 |
| name      | ResellerAdmin                    |
+-----------+----------------------------------+
root@controller:~# openstack role add --user testreseller --project testreseller ResellerAdmin
root@controller:~#
```

验证一下，是否 ResellerAdmin 加上了

```shell
root@controller:~# cat admin-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@controller:~# 
root@controller:~# . admin-openrc 
root@controller:~# openstack role assignment list --role ResellerAdmin
+----------------------------------+----------------------------------+-------+----------------------------------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+-----------+
| 1671364ac7a945e5ae164e100de2c366 | c83158ab5c3348ab9d148673c23903be |       | 3320a273a7094d789f99756b0e9d0f4f |        | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+-----------+
root@controller:~# openstack user show  c83158ab5c3348ab9d148673c23903be
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| default_project_id  | 3320a273a7094d789f99756b0e9d0f4f |
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c83158ab5c3348ab9d148673c23903be |
| name                | testreseller                     |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
root@controller:~# 
```
确实是加上了。

给 testreseller 这个 accout 限制 大小

```shell
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>       --os-project-name testreseller --os-project-domain-name default \
>       --os-username testreseller --os-user-domain-name default \
>       --os-password openstack post -m quota-bytes:10000 
root@controller:~# 
```

查看一下用户 admin 与 testreseller 的区别。

```shell
root@controller:~# cat admin-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@controller:~# . admin-openrc 
root@controller:~# swift stat -v
                     StorageURL: http://controller:8080/v1/AUTH_f04ec0abf3d1460dad82608bb03af589
                     Auth Token: gAAAAABcSZpeeHt6OUARCkWhJaVQ-1lOWxqxscVYMl1eAnrmfwXk40injCPY7V0ZjQOv7wReLgjv4Z4qM_ZzDlNK-KRz2AQFLTVxe4PdQJk7dEazzzQ-MTpMtD2WEqrYfgxdtbEXLE3Iw9Tthhbn-XHnc2OM3sJdIsTVBnvhDc4Tf4NBnItIQ2c
                        Account: AUTH_f04ec0abf3d1460dad82608bb03af589
                     Containers: 1
                        Objects: 1
                          Bytes: 26262280
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 26262280
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txc6f0ed7792c546bfbd160-005c499a5e
                    X-Timestamp: 1547641627.86782
                     X-Trans-Id: txc6f0ed7792c546bfbd160-005c499a5e
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
root@controller:~# cat testreseller-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=testreseller
export OS_USERNAME=testreseller
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@controller:~# . testreseller-openrc 
root@controller:~# swift stat -v
                 StorageURL: http://controller:8080/v1/AUTH_3320a273a7094d789f99756b0e9d0f4f
                 Auth Token: gAAAAABcSY0Uyh27qJWIkueCQZ6ZWhXP-NJbGj4dJ6qRefiYEqBoT_9SfR86cyucBRPxL3dxOlwy-5mHtZ0M8lE-TveWKbLijW3rJDNolFEQria_tn9rdE4q5SK5a5Rl6LhF7XxBaTBb5cXFRozFcivAi8RKbfgqCX6mLpFnupEWfc_Dh5JwDCA
                    Account: AUTH_3320a273a7094d789f99756b0e9d0f4f
                 Containers: 0
                    Objects: 0
                      Bytes: 0
           Meta Quota-Bytes: 10000
X-Account-Project-Domain-Id: default
     X-Openstack-Request-Id: txa424375b986a4b82a61e4-005c498d14
                X-Timestamp: 1548323944.28538
                 X-Trans-Id: txa424375b986a4b82a61e4-005c498d14
               Content-Type: application/json; charset=utf-8
              Accept-Ranges: bytes
root@controller:~# 
```

观察到：

- admin 用户不受影响
- testreseller 用户有多一行`Meta Quota-Bytes: 10000` 也就是说，限制testreseller 用户的存储大小为 10000 bytes


### check

chrome打开： http://192.168.0.50/horizon/project/containers/container/lt10000

但是，发现，上传了 4M, 30M, 70M, 112M 的文件，都可以。为什么？？？

### 问题解决了

从 https://blog.csdn.net/my_vips/article/details/17919167 看到下面这一句话：

```
使用admin为test租户设置限额,test租户的url地址为:http://192.168.26.69:8080/v1/AUTH_3e2a0a2df18b4f86a52e2dc6ad3cb989
swift -V 2 -A http://192.168.26.69:5000/v2.0 -U admin:admin -K ADMIN_PASS --os-storage-url=http://192.168.26.69:8080/v1/AUTH_3e2a0a2df18b4f86a52e2dc6ad3cb989 post -m quota-bytes:100000000
```

原来，不是自己给自己配置，而是通过具体 ResellerAdmin 角色的用户给 其它用户 配置 quota-bytes.

#### (可不看)之前的错误尝试

```shell
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name demo --os-project-domain-name default \
>   --os-username demo --os-user-domain-name default \
>   --os-password openstack post -m quota-bytes:100
Account POST failed: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 403 Forbidden  [first 60 chars of response] <html><h1>Forbidden</h1><p>Access was denied to this resourc
Failed Transaction ID: txfa9b8b212b504f10af48c-005c49a542
root@controller:~# . admin-openrc 
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3  --os-project-name demo --os-project-domain-name default   --os-username demo --os-user-domain-name default   --os-password openstack post -m quota-bytes:100
Account POST failed: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 403 Forbidden  [first 60 chars of response] <html><h1>Forbidden</h1><p>Access was denied to this resourc
Failed Transaction ID: tx2814b6402eba42e39c8a1-005c49a55b
root@controller:~# 
```

### 把 admin 用户赋予 ResellerAdmin 

```shell
root@controller:~# openstack role add --user admin --project admin ResellerAdmin
root@controller:~# openstack project list
+----------------------------------+--------------+
| ID                               | Name         |
+----------------------------------+--------------+
| 3320a273a7094d789f99756b0e9d0f4f | testreseller |
| 7d6eaa90d74a4f239963933c3a744df3 | demo         |
| a640c74e595c44c4902d1c5ebc3afa8a | service      |
| f04ec0abf3d1460dad82608bb03af589 | admin        |
+----------------------------------+--------------+
root@controller:~# openstack role add --user admin ResellerAdmin
Must specify either system, domain, or project
root@controller:~#
```

demo租户的url地址为: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3(这个地址可以有多种方式拿到，最简单的，就是访问：http://192.168.0.50/horizon/project/api_access/)

那么使用admin为demo租户设置限额200bytes, 则设置如下：

```shell
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name demo --os-project-domain-name default \
>   --os-username demo --os-user-domain-name default \
>   --os-password openstack stat
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 8
                        Objects: 51
                          Bytes: 8239369253
Containers in policy "policy-0": 8
   Objects in policy "policy-0": 51
     Bytes in policy "policy-0": 8239369253
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txd1e045cb10584e7da8e5f-005c49a53b
                    X-Timestamp: 1546928904.74292
                     X-Trans-Id: txd1e045cb10584e7da8e5f-005c49a53b
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
root@controller:~# swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 post -m quota-bytes:200
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name demo --os-project-domain-name default \
>   --os-username demo --os-user-domain-name default \
>   --os-password openstack stat
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 8
                        Objects: 51
                          Bytes: 8239369253
Containers in policy "policy-0": 8
   Objects in policy "policy-0": 51
     Bytes in policy "policy-0": 8239369253
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx1baa9f71649a4f5f9a5e0-005c49a63a
                    X-Timestamp: 1546928904.74292
                     X-Trans-Id: tx1baa9f71649a4f5f9a5e0-005c49a63a
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

成功了。

#### check

http://192.168.0.50/horizon/project/containers/container/lt200

因为之前的account 的总大小已经大于200，所以，哈哈，现在连4K的文件都上传不了。

### 去除 quota-bytes

```shell
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name demo --os-project-domain-name default \
>   --os-username demo --os-user-domain-name default \
>   --os-password openstack stat
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 8
                        Objects: 51
                          Bytes: 8239369253
Containers in policy "policy-0": 8
   Objects in policy "policy-0": 51
     Bytes in policy "policy-0": 8239369253
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx1baa9f71649a4f5f9a5e0-005c49a63a
                    X-Timestamp: 1546928904.74292
                     X-Trans-Id: tx1baa9f71649a4f5f9a5e0-005c49a63a
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 post -m quota-bytes:
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3  --os-project-name demo --os-project-domain-name default   --os-username demo --os-user-domain-name default   --os-password openstack stat
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 9
                        Objects: 51
                          Bytes: 8239369253
Containers in policy "policy-0": 9
   Objects in policy "policy-0": 51
     Bytes in policy "policy-0": 8239369253
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx39a80777df094e57800d4-005c49a6c2
                    X-Timestamp: 1546928904.74292
                     X-Trans-Id: tx39a80777df094e57800d4-005c49a6c2
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```
#### check

http://192.168.0.50/horizon/project/containers/container/lt200

上传成功。

## ref

- https://www.ibm.com/developerworks/community/forums/html/topic?id=813eb4fd-c0c6-43a5-9317-a35e4081ef72
- https://docs.openstack.org/swift/latest/middleware.html#module-swift.common.middleware.account_quotas
- https://docs.openstack.org/swift/latest/middleware.html#module-swift.common.middleware.container_quotas
- https://docs.openstack.org/swift/latest/#common-configuration
- https://docs.openstack.org/keystone/rocky/admin/cli-manage-projects-users-and-roles.html
- https://blog.csdn.net/my_vips/article/details/17919167
- https://blog.csdn.net/dysj4099/article/details/8941465
- https://blog.csdn.net/my_vips/article/details/25878751
- https://developer.openstack.org/api-ref/object-store/?expanded=#create-update-or-delete-container-metadata
- https://docs.openstack.org/swift/latest/api/container_quotas.html
