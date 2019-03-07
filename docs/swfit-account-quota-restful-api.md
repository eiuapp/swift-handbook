
通过 RESTFUL API 调用实现 account的 限额


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

Assumptions:

- swift使用ResellerAdmin角色来设置account的quota
- KeyStone V3 is setup and Swift is configured. 
- Python openstack client command line interface is available.
- By default, 'admin' user and 'admin' project is created and assigned to 'default' Domain.  We will use this in this exercise, however you can configure different user, project and domain to accomplish ResellerAdmin setup.
- Openrc file is available to work with openstack CLI client.
 


## 注意

- 开启 ResellerAdmin 角色
- 不是admin给admin配置，而是 ResellerAdmin 角色的用户给 其它用户 配置 quota.

## step

### check accout quota 生效

```shell
root@controller:~# . admin-openrc
root@controller:~# swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c post -m quota-bytes:14568
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3  --os-project-name ptest1 --os-project-domain-name default   --os-username utest1 --os-user-domain-name default   --os-password openstack stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 7
                          Bytes: 108071
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 7
     Bytes in policy "policy-0": 108071
               Meta Quota-Bytes: 14568
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx92ec60cc71c647a0b8522-005c4bceeb
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: tx92ec60cc71c647a0b8522-005c4bceeb
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

### 构造 RESTFUL API

#### 思路

因为我们这里是要修改 account 的 quota 

通过 https://docs.openstack.org/swift/latest/middleware.html#module-swift.common.middleware.account_quotas 中的

```shell
swift -A http://127.0.0.1:8080/auth/v1.0 -U account:reseller -K secret post -m quota-bytes:10000
```

我们知道：

- 发送的是post请求
- "-m quota-bytes:" ，修改的是metadata

那就去找 object-store API 文档 https://developer.openstack.org/api-ref/object-store/index.html ，找到 Account ，找到 POST. 
也就是查找到： https://developer.openstack.org/api-ref/object-store/index.html?expanded=create-update-or-delete-account-metadata-detail 

比如，Update account metadata:

```shell
curl -i $publicURL -X POST -H "X-Auth-Token: $token" -H "X-Account-Meta-Subject: AmericanLiterature"
```

展开 ，可以搜索 quota ，然后，找到 `X-Account-Meta-Quota-Bytes (Optional)	header	string	If present, this is the limit on the total size in bytes of objects stored in the account. Typically this value is set by an administrator.` 。看一下大意，应该 `X-Account-Meta-Quota-Bytes` 就是我们想要的。

那么，对于我们的目标，quota, 就是把 

```shell
swift -V 3 -A http://controller:5000/v3 -U admin:admin -K openstack --os-storage-url=http://controller:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c post -m quota-bytes:14568
```

转成下面语句：

```shell
curl -i $publicURL -X POST -H "X-Auth-Token: $token" -H "X-Account-Meta-Quota-Bytes: 14568"
```

下面，我们就先去找 publicURL 和 token。

当然，这里的 token ,就是 使用 admin 的 token了。

#### 拿 publicURL

这里我们依然是修改 utest1 的 quota

utest1 的 http://192.168.0.50:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c

#### 拿 token

这里使用 admin 用户的 token

```shell
root@controller:~# . admin-openrc 
root@controller:~# swift stat -v | grep Token
                     Auth Token: gAAAAABcS9g0300q8mc5Hb9cakWIxkDDVTC4HwkwQ2TMsXP3jH0CkFYMuMlMeJ1b2pmx04GQ822oN_goj7AiAeWspEFn0W5vg9WuDLkvEuANAMPpX6TzlHIg9trNglZBqPxdczpOO33FrKifAtqYksNs3Sdbj6xFcgzrltXrwkuf6vdM13JUzkA
root@controller:~# 
```

#### curl 调用 RESTFUL

##### POST

```shell
root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3  --os-project-name ptest1 --os-project-domain-name default   --os-username utest1 --os-user-domain-name default   --os-password openstack stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 7
                          Bytes: 108071
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 7
     Bytes in policy "policy-0": 108071
               Meta Quota-Bytes: 14568
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx6b8de30f245d471cbb2b5-005c4bda73
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: tx6b8de30f245d471cbb2b5-005c4bda73
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# curl -i http://192.168.0.50:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c -X POST -H "X-Auth-Token: gAAAAABcS9g0300q8mc5Hb9cakWIxkDDVTC4HwkwQ2TMsXP3jH0CkFYMuMlMeJ1b2pmx04GQ822oN_goj7AiAeWspEFn0W5vg9WuDLkvEuANAMPpX6TzlHIg9trNglZBqPxdczpOO33FrKifAtqYksNs3Sdbj6xFcgzrltXrwkuf6vdM13JUzkA" -H "X-Account-Meta-Quota-Bytes: 123456"
HTTP/1.1 204 No Content
Content-Length: 0
Content-Type: text/html; charset=UTF-8
X-Trans-Id: tx87763d360ed5495eae4a7-005c4bda99
X-Openstack-Request-Id: tx87763d360ed5495eae4a7-005c4bda99
Date: Sat, 26 Jan 2019 03:57:13 GMT

root@controller:~# swift --os-auth-url http://controller:5000/v3  --auth-version 3  --os-project-name ptest1 --os-project-domain-name default   --os-username utest1 --os-user-domain-name default   --os-password openstack stat
                        Account: AUTH_e8ed1722599643b5802a322341b4e02c
                     Containers: 1
                        Objects: 7
                          Bytes: 108071
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 7
     Bytes in policy "policy-0": 108071
               Meta Quota-Bytes: 123456
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txc44a58507f4d4b76b07dc-005c4bda9c
                    X-Timestamp: 1548334795.35484
                     X-Trans-Id: txc44a58507f4d4b76b07dc-005c4bda9c
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

好了，这样就把 utest1 在 ptest1 和 Default 中的quota 设置为 123456 bytes 了。

##### HEAD

同理，如果你想要获取 用户 的 metadata，则参考： https://developer.openstack.org/api-ref/object-store/index.html?expanded=show-account-metadata-detail#show-account-metadata

```shell
root@controller:~# curl -i http://192.168.0.50:8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c -X HEAD -H "X-Auth-Token: gAAAAABcS_rsWTz2Qhz931Wc-oL7c9QwOvHsTXOKJ3RY6A83HX90MbZ0fhrg2b3Yt_PpXwNgt_SQKHO5RS6HnixW9MAVE0TvSmEcGHn-P8rQrJD1auweHFLC_Jl_hPRcYvRf1Mtl802MFwxuIbevAhd2tEu8q-D5seqZbBgBP8R7HhD6JIlTGdE" 
Warning: Setting custom HTTP method to HEAD with -X/--request may not work the 
Warning: way you want. Consider using -I/--head instead.
HTTP/1.1 204 No Content
Content-Length: 0
X-Account-Container-Count: 1
X-Account-Object-Count: 7
X-Account-Storage-Policy-Policy-0-Bytes-Used: 108071
X-Account-Storage-Policy-Policy-0-Container-Count: 1
X-Timestamp: 1548334795.35484
X-Account-Storage-Policy-Policy-0-Object-Count: 7
X-Account-Bytes-Used: 108071
X-Account-Meta-Quota-Bytes: 1234567
Content-Type: application/json; charset=utf-8
Accept-Ranges: bytes
x-account-project-domain-id: default
X-Trans-Id: tx0924923381ba4b13aa118-005c4bfafd
X-Openstack-Request-Id: tx0924923381ba4b13aa118-005c4bfafd
Date: Sat, 26 Jan 2019 06:15:25 GMT

root@controller:~# 
```

上面返回结果中的 `X-Account-Meta-Quota-Bytes` 就是我们想要的。

#### python 调用 RESTFUL

啥也不说了，直接上代码：

```python
import requests
import json
URL = 'http://192.168.0.50:5000/v3/auth/tokens'
body = {
        "auth": {
                    "identity": {
                                    "methods": [
                                                        "password"
                                                    
                                    ],
                                    "password": {
                                                        "user": {
                                                            "name": "admin",
                                            "domain": {
                            "name": "Default"
                                
                                            },
                                            "password": "openstack"
                                                
                                                        }
                                                    
                                    }
                                
                    },
                    "scope": {
                                    "project": {
                                                        "domain": {
                                                                                "name": "Default"
                                                                            
                                                        },
                                                        "name": "admin"
                                                    
                                    }
                                
                    }
                
        }
    
}
body = json.dumps(body)
headers = {'Content-Type':'application/json'}
res = requests.post(URL,data=body,headers=headers)
token =res.headers['X-Subject-Token']

# utest1 storage URL
swiftproxy="192.168.0.50"
storageURL = "http://" + swiftproxy + ":8080/v1/AUTH_e8ed1722599643b5802a322341b4e02c"

# set 
quotabytes = "654321"
headers = {'X-Auth-Token':token,"Content-Type": 'application/json', "X-Account-Meta-Quota-Bytes": quotabytes}
res = requests.post(storageURL,headers=headers)
print(res.status_code)

# get 
headers = {'X-Auth-Token':token,"Content-Type": 'application/json'}
res_get = requests.get(storageURL,headers=headers)
print(res_get.headers['X-Account-Meta-Quota-Bytes'])
print(res_get.text)
```

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
