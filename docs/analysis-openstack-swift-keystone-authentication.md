+++
title = "SWIFt 通过 keystone 认证，并实现调用swift接口"
date = 2018-11-23T00:00:00-08:00
lastmod = 2019-01-18T02:11:47-08:00
tags = ["openstack", "swift"]
categories = ["openstack"]
draft = false
weight = 3001
+++

## 总体 {#总体}

1.  安装 keystone 认证
2.  调用swift接口
    1.  向keystone发送请求。得到X-Auth-Token
    2.  带X-Auth-Token向swift服务发出请求


## env {#env}

1.  controller node 上安装了 keystone, swift-proxy 服务。
2.  storage node 安装 account, container, object 服务。

ip:

-   controller node:
    -   192.168.0.50
-   storage node:
    -   192.168.0.198
    -   192.168.0.134
    -   192.168.0.135


## 安装 keystone 认证 {#安装-keystone-认证}

swift 与 keystone 结合的配置文档在
<https://docs.openstack.org/swift/latest/overview%5Fauth.html>

文档最后
<https://docs.openstack.org/swift/latest/overview%5Fauth.html#troubleshooting-tips-for-keystoneauth-deployment>
指出了一个验证方式

```shell
swift --os-auth-url https://api.example.com/v3 --auth-version 3\
     --os-project-name project1 --os-project-domain-name domain1 \
     --os-username user --os-user-domain-name domain1 \
     --os-password password list
```

当然这个是按照配置来的。因为，我这里面swift对应的password是 openstack, 所以，我这里的验证如下：

```shell
root@controller:~# openstack --os-identity-api-version=3 --os-auth-url=http://keystonehost:5000/     --os-username=swift --os-user-domain-id=default     --os-project-name=service --os-project-domain-id=default     --os-password=openstack catalog show object-store
+-----------|-----------------------------------------------------------------------------+
| Field     | Value                                                                       |
+-----------|-----------------------------------------------------------------------------+
| endpoints | RegionOne                                                                   |
|           |   public: http://controller:8080/v1/AUTH_a640c74e595c44c4902d1c5ebc3afa8a   |
|           | RegionOne                                                                   |
|           |   admin: http://controller:8080/v1                                          |
|           | RegionOne                                                                   |
|           |   internal: http://controller:8080/v1/AUTH_a640c74e595c44c4902d1c5ebc3afa8a |
|           |                                                                             |
| id        | e929673efa1a4acb9adc4a06e4f56a31                                            |
| name      | swift                                                                       |
| type      | object-store                                                                |
+-----------|-----------------------------------------------------------------------------+
root@controller:~#
```

这里看出，配置是没有问题的。


## 调用swift接口 {#调用swift接口}


### 准备工作 {#准备工作}


#### endpoint {#endpoint}

```shell
root@controller:~# openstack endpoint list
Missing value auth-url required for auth plugin password
root@controller:~# . admin-openrc
root@controller:~# openstack endpoint list
+----------------------------------|-----------|--------------|--------------|---------|-----------|-----------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                           |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-----------------------------------------------+
| 014c4ed5c42042c394d62a1194bf07ce | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3/                    |
| 0b3dc5788cac4f2cb66edc3efacf10c0 | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3/                    |
| 19e727c1668c4e8aae4315e13057fbbd | RegionOne | cinderv2     | volumev2     | True    | public    | http://controller:8776/v2/%(project_id)s      |
| 216d8d8f353a49f78aa4421ff6e8c27b | RegionOne | swift        | object-store | True    | public    | http://controller:8080/v1/AUTH_%(project_id)s |
| 2548787f06524287b6a27d0a562da375 | RegionOne | glance       | image        | True    | internal  | http://controller:9292                        |
| 295cfee33ff04ab7890a84b47e95bc3f | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3/                    |
| 50d3b61b60654a65bda26584b4fe0896 | RegionOne | neutron      | network      | True    | admin     | http://controller:9696                        |
| 55367bdc8db6438da3a4ed82b1a1b04c | RegionOne | glance       | image        | True    | public    | http://controller:9292                        |
| 5b60dcd8e374410f83e675201d76f06f | RegionOne | placement    | placement    | True    | admin     | http://controller:8778                        |
| 610f89fa29764f7eb5c91e737ad0110e | RegionOne | swift        | object-store | True    | admin     | http://controller:8080/v1                     |
| 76d61f360fc140a58b4258c344ea9ae5 | RegionOne | nova         | compute      | True    | internal  | http://controller:8774/v2.1                   |
| 7eece3971f9f4105b031214666940d54 | RegionOne | placement    | placement    | True    | public    | http://controller:8778                        |
| 8237211a85314e0c914ea34851e324b7 | RegionOne | cinderv2     | volumev2     | True    | admin     | http://controller:8776/v2/%(project_id)s      |
| 99b6673fda7f48bb8b6ae11a87ba0e1b | RegionOne | glance       | image        | True    | admin     | http://controller:9292                        |
| a39982358d3041e0aebd7a1a1fb691bb | RegionOne | swift        | object-store | True    | internal  | http://controller:8080/v1/AUTH_%(project_id)s |
| a61cbe1448284dd482725fb3067fa25c | RegionOne | nova         | compute      | True    | public    | http://controller:8774/v2.1                   |
| cfb1df03d1bd42719dd506e0fbee7e5a | RegionOne | cinderv3     | volumev3     | True    | internal  | http://controller:8776/v3/%(project_id)s      |
| cfe6b374e86048c98951d079a3cdaa42 | RegionOne | placement    | placement    | True    | internal  | http://controller:8778                        |
| d9bb0785c23143ed855d0bb74dfe53bb | RegionOne | cinderv3     | volumev3     | True    | public    | http://controller:8776/v3/%(project_id)s      |
| e1f98d0e79f549a5bc46bcf05705c4b1 | RegionOne | neutron      | network      | True    | public    | http://controller:9696                        |
| f089ebe1cb5040319e942dbc607e9930 | RegionOne | cinderv3     | volumev3     | True    | admin     | http://controller:8776/v3/%(project_id)s      |
| f3350a1d66fa4813a091beef3782b6fd | RegionOne | neutron      | network      | True    | internal  | http://controller:9696                        |
| f5460d92ef8b428b9dc354628acdb289 | RegionOne | cinderv2     | volumev2     | True    | internal  | http://controller:8776/v2/%(project_id)s      |
| fb5c30679ed14e078646041e75e77294 | RegionOne | nova         | compute      | True    | admin     | http://controller:8774/v2.1                   |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-----------------------------------------------+
root@controller:~#
```

从 openstack endpoint 拿到 swift 的 URL 是： <http://controller:8080/v1>


#### 试验swift 状态 {#试验swift-状态}

```shell
root@controller:~# swift stat -v
                     StorageURL: http://controller:8080/v1/AUTH_f04ec0abf3d1460dad82608bb03af589
                     Auth Token: gAAAAABcQUnrEMgo3fTeFVMWrYcA6-6t6XNAGljTzPNdH111cbawcBjozo0DUvHhsOY4hucyfzhBAzwMlzsigyj1jOqdQndn_6cO5k8JTrQtyYGzLxJ5weEdHnV6HVszgQBtyxq17ut1_RTu_DhVT-m5MLKpMGcglszF-XCcC-Ua9I2NHTLicCg
                        Account: AUTH_f04ec0abf3d1460dad82608bb03af589
                     Containers: 1
                        Objects: 1
                          Bytes: 26262280
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 26262280
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx753c958eb6ad4bd5b2ef0-005c4149eb
                    X-Timestamp: 1547641627.86782
                     X-Trans-Id: tx753c958eb6ad4bd5b2ef0-005c4149eb
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~#
```

从 swift stat -v 的返回可以看到 Auth Token 且状态正常。
所以，是可以使用 curl, python 通过RESTFUL API调用swift服务的。


#### 通过swift命令，获得请求参数 {#通过swift命令-获得请求参数}

```shell
root@controller:~# swift stat -v
                     StorageURL: http://controller:8080/v1/AUTH_f04ec0abf3d1460dad82608bb03af589
                     Auth Token: gAAAAABcQUnrEMgo3fTeFVMWrYcA6-6t6XNAGljTzPNdH111cbawcBjozo0DUvHhsOY4hucyfzhBAzwMlzsigyj1jOqdQndn_6cO5k8JTrQtyYGzLxJ5weEdHnV6HVszgQBtyxq17ut1_RTu_DhVT-m5MLKpMGcglszF-XCcC-Ua9I2NHTLicCg
                        Account: AUTH_f04ec0abf3d1460dad82608bb03af589
                     Containers: 1
                        Objects: 1
                          Bytes: 26262280
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 26262280
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx753c958eb6ad4bd5b2ef0-005c4149eb
                    X-Timestamp: 1547641627.86782
                     X-Trans-Id: tx753c958eb6ad4bd5b2ef0-005c4149eb
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# swift --help
usage: swift [--version] [--help] [--os-help] [--snet] [--verbose]

Command-line interface to the OpenStack Swift API.

Positional arguments:

Examples:
  swift download --help

  swift -A https://api.example.com/v1.0 \
      -U user -K api_key stat -v

  swift --os-auth-url https://api.example.com/v2.0 \
      --os-tenant-name tenant \
      --os-username user --os-password password list

  swift --os-auth-url https://api.example.com/v3 --auth-version 3\
      --os-project-name project1 --os-project-domain-name domain1 \
      --os-username user --os-user-domain-name domain1 \
      --os-password password list

  swift --os-auth-url https://api.example.com/v3 --auth-version 3\
      --os-project-id 0123456789abcdef0123456789abcdef \
      --os-user-id abcdef0123456789abcdef0123456789 \
      --os-password password list

  swift --os-auth-token 6ee5eb33efad4e45ab46806eac010566 \
      --os-storage-url https://10.1.5.2:8080/v1/AUTH_ced809b6a4baea7aeab61a \
      list

  swift list --lh

optional arguments:
root@controller:~#
```

通过这个example 可以看到，原来之前的 swift stat -v 就是因为使用了 . admin-openrc 中的参数。

因为，后续，我们是不会使用 admin 这个帐号的，所以，现在起我们切换成 demo 帐号。

```shell
root@controller:~# . demo-openrc
root@controller:~# openstack endpoint list
You are not authorized to perform the requested action: identity:list_endpoints. (HTTP 403) (Request-ID: req-b7efc3d8-0046-4847-b4ed-2e99d53d9e35)
root@controller:~# swift stat -v
                     StorageURL: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Auth Token: gAAAAABcQUz9FSJBIUSo2DC5sOCFLcytonKVdX6MOf6m9KY4-IcGlTztCZkULlkUKhaiRzCdYKJeNE06MblaIjArbb-VASeMelaKL3SBRWTdRn2Qb3TygrmRt3CFuU2bRQGitjuxp4m8WY7hYroL97cSyrDKTpCHKYhfz9X05F2yFlQ-gLZqC7w
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 6
                        Objects: 29
                          Bytes: 7319499838
Containers in policy "policy-0": 6
   Objects in policy "policy-0": 29
     Bytes in policy "policy-0": 7319499838
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: tx85bf81afca4b40bdbd431-005c414cfd
                    X-Timestamp: 1546928904.74292
                     X-Trans-Id: tx85bf81afca4b40bdbd431-005c414cfd
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~#
```

从上面的结果看到，demo 用户是不能访问 endpoint 的。但是，可以访问到 swift 服务。


#### 构造 swift example {#构造-swift-example}

看一下 demo-openrc 中的内容。

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
root@controller:~#
```

按 swift example 的方式配置如下。

注意了：下面的这个命令，是可以不预先运行\`. demo-openrc\`命令的。

```shell
ubuntu@controller:~$  swift --os-auth-url http://controller:5000/v3 --auth-version 3\
>       --os-project-name demo --os-project-domain-name Default \
>       --os-username demo --os-user-domain-name Default \
>       --os-password openstack list
container1
container1_segments
container2
jinweilai-work
pub1
test3
ubuntu@controller:~$
```

如果像下面的这样，把project 的信息去除，则会报错

```shell
ubuntu@controller:~$  swift --os-auth-url http://controller:5000/v3 --auth-version 3\
>       --os-username demo --os-user-domain-name Default \
>       --os-password openstack list
No project name or project id specified.
ubuntu@controller:~$
```

说明每一个地方的设置，都要加到请求参数中去。

也就是这个 demo-openrc 是有scope 的。


#### 分析swift example {#分析swift-example}

那么，上面的命令是如何实现查看出 demo 这个帐号的 container 信息的呢？

根据文档知道，是先向 keystone 拿到 X-Auth-Token, 然后拿上这个X-Auth-Token 向swift 请求的。

那这样子说，我们可以用 \`swift stat -v\` 中的 \`Auth Token\` 直接向swift请求，试一下，是不是有效。（其实通过keystone日志/var/log/keystone/keystone-wsgi-public.log 和 swift日志（存放在系统日志）/var/log/syslog , 可以知道，就是这样的。）

让我们试一下。

在 controller node 中 使用 \`swift stat -v\`再来一次。

```shell
root@controller:~# swift stat -v
                     StorageURL: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Auth Token: gAAAAABcQVk7Q6QgnbTmTobnt36rLZmifbCMow4W-MWfQ9txcRyHEYOPXWRKAcc2r7bAVG0uS_VNC5GyIPw6FjQx3Bb-mofESZDEPs5AHe8m2Pg1Nwfmhrd8lg_4VqWZRffUQIrrRiNH1JSViBXFRZJn0zwSdwUREoskIuetQ0uZ5FWXuQOz240
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 6
                        Objects: 29
                          Bytes: 7319499838
Containers in policy "policy-0": 6
   Objects in policy "policy-0": 29
     Bytes in policy "policy-0": 7319499838
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txb40ee72900e8487a8e5ec-005c41593b
                    X-Timestamp: 1546928904.74292
                     X-Trans-Id: txb40ee72900e8487a8e5ec-005c41593b
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~#
```

然后根据
<https://developer.openstack.org/api-ref/object-store/index.html?expanded=show-account-details-and-list-containers-detail#show-container-details-and-list-objects>
去任何一台公网机器上

```shell
➜  swift curl -i "http://192.168.0.50:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json" -X GET -H "X-Auth-Token: gAAAAABcQVk7Q6QgnbTmTobnt36rLZmifbCMow4W-MWfQ9txcRyHEYOPXWRKAcc2r7bAVG0uS_VNC5GyIPw6FjQx3Bb-mofESZDEPs5AHe8m2Pg1Nwfmhrd8lg_4VqWZRffUQIrrRiNH1JSViBXFRZJn0zwSdwUREoskIuetQ0uZ5FWXuQOz240"
HTTP/1.1 200 OK
Content-Length: 337
X-Account-Object-Count: 29
X-Account-Storage-Policy-Policy-0-Bytes-Used: 7319499838
X-Account-Storage-Policy-Policy-0-Container-Count: 6
X-Timestamp: 1546928904.74292
X-Account-Storage-Policy-Policy-0-Object-Count: 29
X-Account-Bytes-Used: 7319499838
X-Account-Container-Count: 6
Content-Type: application/json; charset=utf-8
Accept-Ranges: bytes
x-account-project-domain-id: default
X-Trans-Id: txcd8c653a027b4a5d9a9a0-005c415a1d
X-Openstack-Request-Id: txcd8c653a027b4a5d9a9a0-005c415a1d
Date: Fri, 18 Jan 2019 04:46:21 GMT

[{"count": 9, "bytes": 506788650, "name": "container1"}, {"count": 4, "bytes": 3460191876, "name": "container1_segments"}, {"count": 1, "bytes": 18290688, "name": "container2"}, {"count": 1, "bytes": 375626779, "name": "jinweilai-work"}, {"count": 1, "bytes": 399336, "name": "pub1"}, {"count": 13, "bytes": 2958202509, "name": "test3"}]%
➜  swift
```

得到结果了。哈哈。

那这样子说，我们只要得到正确的\`X-Auth-Token\`，就一定能调用swift服务了。

下面就是依据我们现有的材料，向keystone发送请求, 得到我们要的 X-Auth-Token 了。


### 向keystone发送请求。得到X-Auth-Token {#向keystone发送请求-得到x-auth-token}

根据下面文档
<https://developer.openstack.org/api-ref/identity/v3/index.html?expanded=password-authentication-with-scoped-authorization-detail#identity-api-operations>

结合我们 demo-openrc 和 刚刚的 swift example, 我们得到

生成JSON格式请求body:

```json
{
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                "name": "demo",
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
                "name": "demo"
            }
        }
    }
}
```

向 keystone 的 endpoint

```text
| 0b3dc5788cac4f2cb66edc3efacf10c0 | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3/                    |
```

URL: <http://controller:5000/v3/>

发送请求。得到response. 其中响应 headers 中的 'X-Subject-Token', 就是 X-Auth-Token 了。


### 带X-Auth-Token向swift服务发出请求 {#带x-auth-token向swift服务发出请求}

这一步就是重复刚刚讲过的内容，向swift请求了.
然后根据
<https://developer.openstack.org/api-ref/object-store/index.html?expanded=show-account-details-and-list-containers-detail#show-container-details-and-list-objects>
去任何一台公网机器上

```shell
➜  swift curl -i "http://192.168.0.50:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json" -X GET -H "X-Auth-Token: gAAAAABcQVk7Q6QgnbTmTobnt36rLZmifbCMow4W-MWfQ9txcRyHEYOPXWRKAcc2r7bAVG0uS_VNC5GyIPw6FjQx3Bb-mofESZDEPs5AHe8m2Pg1Nwfmhrd8lg_4VqWZRffUQIrrRiNH1JSViBXFRZJn0zwSdwUREoskIuetQ0uZ5FWXuQOz240"
HTTP/1.1 200 OK
Content-Length: 337
X-Account-Object-Count: 29
X-Account-Storage-Policy-Policy-0-Bytes-Used: 7319499838
X-Account-Storage-Policy-Policy-0-Container-Count: 6
X-Timestamp: 1546928904.74292
X-Account-Storage-Policy-Policy-0-Object-Count: 29
X-Account-Bytes-Used: 7319499838
X-Account-Container-Count: 6
Content-Type: application/json; charset=utf-8
Accept-Ranges: bytes
x-account-project-domain-id: default
X-Trans-Id: txcd8c653a027b4a5d9a9a0-005c415a1d
X-Openstack-Request-Id: txcd8c653a027b4a5d9a9a0-005c415a1d
Date: Fri, 18 Jan 2019 04:46:21 GMT

[{"count": 9, "bytes": 506788650, "name": "container1"}, {"count": 4, "bytes": 3460191876, "name": "container1_segments"}, {"count": 1, "bytes": 18290688, "name": "container2"}, {"count": 1, "bytes": 375626779, "name": "jinweilai-work"}, {"count": 1, "bytes": 399336, "name": "pub1"}, {"count": 13, "bytes": 2958202509, "name": "test3"}]%
➜  swift
```

得到结果了。哈哈。


### 利用 python 来访问swift {#利用-python-来访问swift}

直接上代码了。

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
                                                            "name": "demo",
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
                                                        "name": "demo"
                                    }
                    }
        }
}
body = json.dumps(body)
print(body)
headers = {'Content-Type':'application/json'}
res = requests.post(URL,data=body,headers=headers)
token = res.headers['X-Subject-Token']
print(token)

URL = 'http://192.168.0.50:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3'
headers = {'X-Auth-Token':token,"Content-Type": 'application/json'}
res = requests.get(URL,headers=headers)
print(res.text)
print(res.status_code)
```
