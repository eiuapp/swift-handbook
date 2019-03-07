
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

- 开启 ResellerAdmin 角色
- 不是admin给admin配置，而是 ResellerAdmin 角色的用户给 其它用户 配置 quota.

## step

### 配置 proxy-swift.conf 

要把 swift proxy 节点的 reseller_admin_role = ResellerAdmin 打开注释，重启服务

```shell
root@controller:~# grep ResellerAdmin -rn /etc/swift/proxy-server.conf  
417:reseller_admin_role = ResellerAdmin
root@controller:~# service swift-proxy restart
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

#### 验证一下，是否 ResellerAdmin 加上了

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
root@controller:~# openstack role assignment list --role ResellerAdmin
+----------------------------------+----------------------------------+-------+----------------------------------+--------+-----------+
| Role                             | User                             | Group | Project                          | Domain | Inherited |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+-----------+
| 1671364ac7a945e5ae164e100de2c366 | 8533cb3873974fa29f03832aef7007ca |       | f04ec0abf3d1460dad82608bb03af589 |        | False     |
+----------------------------------+----------------------------------+-------+----------------------------------+--------+-----------+
root@controller:~# openstack user show 8533cb3873974fa29f03832aef7007ca
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8533cb3873974fa29f03832aef7007ca |
| name                | admin                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
root@controller:~# openstack project show f04ec0abf3d1460dad82608bb03af589
+-------------+-----------------------------------------------+
| Field       | Value                                         |
+-------------+-----------------------------------------------+
| description | Bootstrap project for initializing the cloud. |
| domain_id   | default                                       |
| enabled     | True                                          |
| id          | f04ec0abf3d1460dad82608bb03af589              |
| is_domain   | False                                         |
| name        | admin                                         |
| parent_id   | default                                       |
| tags        | []                                            |
+-------------+-----------------------------------------------+
root@controller:~# 
```
确实是加上了。

### 创建新环境

创建新的project, user, role 来实验

准备语句：

```
openstack project create --description "ptest1" ptest1 --domain default    
openstack user create --project ptest1 --password openstack utest1 
openstack role create rtest1
openstack role add --user utest1 --project ptest1 rtest1
```

开始吧

```shell
root@controller:~# . admin-openrc 
root@controller:~# openstack project create --description "ptest1" ptest1 --domain default    
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | ptest1                           |
| domain_id   | default                          |
| enabled     | True                             |
| id          | e8ed1722599643b5802a322341b4e02c |
| is_domain   | False                            |
| name        | ptest1                           |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
root@controller:~# openstack user create --project ptest1 --password openstack utest1 
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| default_project_id  | e8ed1722599643b5802a322341b4e02c |
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 5b8f1fbc7a4548b78b4c59b47ac43e77 |
| name                | utest1                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
root@controller:~# openstack role add --user utest1 --project ptest1 user
```

#### check

```shell
root@controller:~# openstack role list --user utest1 --project ptest1 
Listing assignments using role list is deprecated. Use role assignment list --user <user-name> --project <project-name> --names instead.
+----------------------------------+------+---------+--------+
| ID                               | Name | Project | User   |
+----------------------------------+------+---------+--------+
| aa47471c39884d708f552a3b5aa2f067 | user | ptest1  | utest1 |
+----------------------------------+------+---------+--------+
root@controller:~# 
```


#### 查一下 swift stat

```shell
swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name ptest1 --os-project-domain-name default \
>   --os-username utest1 --os-user-domain-name default \
>   --os-password openstack stat
               Account: AUTH_e8ed1722599643b5802a322341b4e02c
            Containers: 0
               Objects: 0
                 Bytes: 0
       X-Put-Timestamp: 1548334374.80379
           X-Timestamp: 1548334374.80379
            X-Trans-Id: tx97bf4a41e2f54021a2602-005c49b525
          Content-Type: text/plain; charset=utf-8
X-Openstack-Request-Id: tx97bf4a41e2f54021a2602-005c49b525
root@controller:~# 
```

或者 写成一个 openrc 文件，也方便

```shell
root@controller:~# cat utest1-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=ptest1
export OS_USERNAME=utest1
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
root@controller:~# . utest1-openrc 
root@controller:~# swift stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 0
                          Bytes: 0
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 0
     Bytes in policy "policy-0": 0
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txc1aecf0f20a04d7795dfb-005c49b7ff
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: txc1aecf0f20a04d7795dfb-005c49b7ff
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```
去 http://192.168.0.50/horizon/project/ 创建 container 吧

http://192.168.0.50/horizon/project/containers/container/lt200

#### (可以不看)下面展示我走过的一段弯路

如果在这个新环境中，创建一个新的role，并赋予新用户utest1, 由于 新role没有添加属性，会导致无法通过 `swift stat` 查询。具体如下：

```shell
root@controller:~# . admin-openrc 
root@controller:~# openstack role create rtest1
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | a541e61be9564648a23f4f2d535e0014 |
| name      | rtest1                           |
+-----------+----------------------------------+
root@controller:~# openstack role add --user utest1 --project ptest1 rtest1
root@controller:~# 
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name ptest1 --os-project-domain-name default \
>   --os-username utest1 --os-user-domain-name default \
>   --os-password openstack stat
Unauthorized. Check username/id, password, tenant name/id and user/tenant domain name/id.
root@controller:~# 
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name ptest1 --os-project-domain-name default \
>   --os-username utest1 --os-user-domain-name default \
>   --os-password openstack stat
Account HEAD failed: http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c 403 Forbidden
Failed Transaction ID: tx6a128832b5a544b7941d0-005c49af6e
root@controller:~# 
root@controller:~# openstack role list --user utest1 --project ptest1
Listing assignments using role list is deprecated. Use role assignment list --user <user-name> --project <project-name> --names instead.
+----------------------------------+--------+---------+--------+
| ID                               | Name   | Project | User   |
+----------------------------------+--------+---------+--------+
| a541e61be9564648a23f4f2d535e0014 | rtest1 | ptest1  | utest1 |
+----------------------------------+--------+---------+--------+
root@controller:~# openstack role remove --user utest1 --project ptest1 rtest1
root@controller:~# 
root@controller:~# openstack role list --user utest1 --project ptest1
Listing assignments using role list is deprecated. Use role assignment list --user <user-name> --project <project-name> --names instead.

root@controller:~# 
```

### 给 utest1 这个 accout 限制 大小

#### utest1租户的url地址

utest1租户的url地址为: http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c

这个地址可以有多种方式拿到，最简单的，就是 输入之前创建时的 project, user, password 信息, 访问：http://192.168.0.50/horizon/project/api_access/ 效果如下：

![openstack-account-api-access](https://res.cloudinary.com/dmtixvmgt/image/upload/v1548335569/openstack-account-api-access_rcrnqj.png)

那么使用admin为demo租户设置限额200bytes, (设置时，需要`. admin-openrc`获取admin权限),则设置如下：

```shell
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3\
>   --os-project-name ptest1 --os-project-domain-name default \
>   --os-username utest1 --os-user-domain-name default \
>   --os-password openstack stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 0
                          Bytes: 0
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 0
     Bytes in policy "policy-0": 0
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx4181f950431446739181e-005c49b765
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: tx4181f950431446739181e-005c49b765
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~#   swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c post -m quota-bytes:200
Unauthorized. Check username/id, password, tenant name/id and user/tenant domain name/id.
root@controller:~# . admin-openrc
root@controller:~# swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c post -m quota-bytes:200
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3  --os-project-name ptest1 --os-project-domain-name default   --os-username utest1 --os-user-domain-name default   --os-password openstack stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 0
                          Bytes: 0
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 0
     Bytes in policy "policy-0": 0
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txb6e7ce54af8d4497a5e18-005c49b78a
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: txb6e7ce54af8d4497a5e18-005c49b78a
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

成功了。

#### check

访问 http://192.168.0.50/horizon/project/containers/container/lt200 。此时，依然可以上传 大于 200 bytes 的文件（别问我为什么？哈哈原因就是这：Due to the eventual consistency further uploads might be possible until the account size has been updated.）。但是过一会，就上传不了。哈哈。说明，真正在生产的时候，这里应该要限制一下时间，等待集群刷新。哈哈。

效果如下：

![openstack-swift-already-have-objects-then-quota-then-unable-to-upload-the-object](https://res.cloudinary.com/dmtixvmgt/image/upload/v1548335569/openstack-swift-already-have-objects-then-quota-then-unable-to-upload-the-object_zykd5w.png)

因为之前的account 的总大小已经大于200，所以，哈哈，现在连4K的文件都上传不了。

当然，你可以试一下，把文件全删除掉，然后，你以为不能上传大于 200 bytes 的文件。NO! 依然可以上传，我就测试过了，上传了 2个 14K 的文件。

```shell
root@controller:~# swift stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 2
                          Bytes: 29880
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 2
     Bytes in policy "policy-0": 29880
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx566d10c4346b4c3ea0c75-005c49bbf5
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: tx566d10c4346b4c3ea0c75-005c49bbf5
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

过了好一会，才更新状态：

```shell
root@controller:~# swift stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 2
                          Bytes: 29880
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 2
     Bytes in policy "policy-0": 29880
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx9018806c738746c6b6d68-005c49bdaf
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: tx9018806c738746c6b6d68-005c49bdaf
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# swift stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 1
                          Bytes: 14940
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 14940
               Meta Quota-Bytes: 200
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx46a6314ab5a8463fb3541-005c49bdc9
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: tx46a6314ab5a8463fb3541-005c49bdc9
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

当然，过了一小会，就真的再也不能上传了。哈哈。


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
root@controller:~#   swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c post -m quota-bytes:
Unauthorized. Check username/id, password, tenant name/id and user/tenant domain name/id.
root@controller:~# . admin-openrc
root@controller:~# swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c post -m quota-bytes:
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
