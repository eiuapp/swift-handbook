报 `Auth versions 2.0 and 3 require python-keystoneclient` 错误

## env 

```
root@controller:~# . admin-openrc 
root@controller:~# swift stat 

Auth versions 2.0 and 3 require python-keystoneclient, install it or use Auth
version 1.0 which requires ST_AUTH, ST_USER, and ST_KEY environment
variables to be set or overridden with -A, -U, or -K.

root@controller:~# 
```

## step

### 检查 python-keystoneclient 是否安装

https://docs.openstack.org/swift/queens/install/controller-install-ubuntu.html

```shell
apt-get install swift swift-proxy python-swiftclient \
  python-keystoneclient python-keystonemiddleware \
  memcached
```

发现已经安装过了。

### swift-proxy 服务是否正常

```
root@controller:/etc/swift# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Wed 2019-03-06 12:55:50 CST; 7s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 25557 ExecStop=/etc/init.d/swift-proxy stop (code=exited, status=0/SUCCESS)
  Process: 25568 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 83.6M
      CPU: 1.101s
   CGroup: /system.slice/swift-proxy.service
           ├─25581 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─25590 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─25591 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─25592 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─25593 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Mar 06 12:55:47 controller proxy-server[25590]: Starting Keystone auth_token middleware
Mar 06 12:55:47 controller proxy-server[25590]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True.
Mar 06 12:55:47 controller proxy-server[25592]: Starting Keystone auth_token middleware
Mar 06 12:55:47 controller proxy-server[25592]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True.
Mar 06 12:55:47 controller proxy-server[25593]: Starting Keystone auth_token middleware
Mar 06 12:55:47 controller proxy-server[25593]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True.
Mar 06 12:55:50 controller swift-proxy[25568]: Starting proxy-server...(/etc/swift/proxy-server.conf)
Mar 06 12:55:50 controller swift-proxy[25568]: No handlers could be found for logger "keystonemiddleware._common.config"
Mar 06 12:55:50 controller swift-proxy[25568]:    ...done.
Mar 06 12:55:50 controller systemd[1]: Started LSB: Swift proxy server.
root@controller:/etc/swift# 
```

### openstack 是否正常

```
root@controller:~# openstack endpoint list
+----------------------------------|-----------|--------------|--------------|---------|-----------|-------------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                             |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-------------------------------------------------+
| 014c4ed5c42042c394d62a1194bf07ce | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3/                      |
| 0b3dc5788cac4f2cb66edc3efacf10c0 | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3/                      |
| 19e727c1668c4e8aae4315e13057fbbd | RegionOne | cinderv2     | volumev2     | True    | public    | http://controller:8776/v2/%(project_id)s        |
| 2548787f06524287b6a27d0a562da375 | RegionOne | glance       | image        | True    | internal  | http://controller:9292                          |
| 295cfee33ff04ab7890a84b47e95bc3f | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3/                      |
| 50d3b61b60654a65bda26584b4fe0896 | RegionOne | neutron      | network      | True    | admin     | http://controller:9696                          |
| 55367bdc8db6438da3a4ed82b1a1b04c | RegionOne | glance       | image        | True    | public    | http://controller:9292                          |
| 5b60dcd8e374410f83e675201d76f06f | RegionOne | placement    | placement    | True    | admin     | http://controller:8778                          |
| 76d61f360fc140a58b4258c344ea9ae5 | RegionOne | nova         | compute      | True    | internal  | http://controller:8774/v2.1                     |
| 7eece3971f9f4105b031214666940d54 | RegionOne | placement    | placement    | True    | public    | http://controller:8778                          |
| 8237211a85314e0c914ea34851e324b7 | RegionOne | cinderv2     | volumev2     | True    | admin     | http://controller:8776/v2/%(project_id)s        |
| 99b6673fda7f48bb8b6ae11a87ba0e1b | RegionOne | glance       | image        | True    | admin     | http://controller:9292                          |
| a61cbe1448284dd482725fb3067fa25c | RegionOne | nova         | compute      | True    | public    | http://controller:8774/v2.1                     |
| a9b074060f2e45f898c388dd382980c3 | RegionOne | swift        | object-store | True    | admin     | http://192.168.0.50:8080/v1/AUTH_%(project_id)s |
| bf59c9ad0d5e4fc5ad47bb9da0171e70 | RegionOne | swift        | object-store | True    | public    | http://192.168.0.50:8080/v1/AUTH_%(project_id)s |
| cfb1df03d1bd42719dd506e0fbee7e5a | RegionOne | cinderv3     | volumev3     | True    | internal  | http://controller:8776/v3/%(project_id)s        |
| cfe6b374e86048c98951d079a3cdaa42 | RegionOne | placement    | placement    | True    | internal  | http://controller:8778                          |
| d58337580bc34b97b83129ead16f46d3 | RegionOne | swift        | object-store | True    | internal  | http://192.168.0.50:8080/v1/AUTH_%(project_id)s |
| d9bb0785c23143ed855d0bb74dfe53bb | RegionOne | cinderv3     | volumev3     | True    | public    | http://controller:8776/v3/%(project_id)s        |
| e1f98d0e79f549a5bc46bcf05705c4b1 | RegionOne | neutron      | network      | True    | public    | http://controller:9696                          |
| f089ebe1cb5040319e942dbc607e9930 | RegionOne | cinderv3     | volumev3     | True    | admin     | http://controller:8776/v3/%(project_id)s        |
| f3350a1d66fa4813a091beef3782b6fd | RegionOne | neutron      | network      | True    | internal  | http://controller:9696                          |
| f5460d92ef8b428b9dc354628acdb289 | RegionOne | cinderv2     | volumev2     | True    | internal  | http://controller:8776/v2/%(project_id)s        |
| fb5c30679ed14e078646041e75e77294 | RegionOne | nova         | compute      | True    | admin     | http://controller:8774/v2.1                     |
+----------------------------------|-----------|--------------|--------------|---------|-----------|-------------------------------------------------+
root@controller:~# 
```
或者

```shell
root@controller:~# openstack --os-identity-api-version=3 --os-auth-url=http://keystonehost:5000/     --os-username=swift --os-user-domain-id=default     --os-project-name=service --os-project-domain-id=default     --os-password=openstack catalog show object-store
+-----------|-------------------------------------------------------------------------------+
| Field     | Value                                                                         |
+-----------|-------------------------------------------------------------------------------+
| endpoints | RegionOne                                                                     |
|           |   admin: http://192.168.0.50:8080/v1/AUTH_a640c74e595c44c4902d1c5ebc3afa8a    |
|           | RegionOne                                                                     |
|           |   public: http://192.168.0.50:8080/v1/AUTH_a640c74e595c44c4902d1c5ebc3afa8a   |
|           | RegionOne                                                                     |
|           |   internal: http://192.168.0.50:8080/v1/AUTH_a640c74e595c44c4902d1c5ebc3afa8a |
|           |                                                                               |
| id        | e929673efa1a4acb9adc4a06e4f56a31                                              |
| name      | swift                                                                         |
| type      | object-store                                                                  |
+-----------|-------------------------------------------------------------------------------+
root@controller:~# 
```

### 检查 keystone 是否正常

https://docs.openstack.org/keystone/rocky/install/keystone-verify-ubuntu.html

```
root@controller:~# unset OS_AUTH_URL OS_PASSWORD
root@controller:~# openstack --os-auth-url http://controller:5000/v3 \
>   --os-project-domain-name Default --os-user-domain-name Default \
>   --os-project-name admin --os-username admin token issue
Password: 
+------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-03-06T07:45:03+0000                                                                                                                                                                |
| id         | gAAAAABcf2xvzNJHdU_36s_NlO_2GFU4NY1CDi1LLEXacDeYHLKLAiZovw4CiTZJ7kVUPhqL_fXRYFrQPNGRU-9bt0WnBrGrpkOBhs-lVJxuyMM7f_MA6jo8UeQDEtryEkLcahejk5wtmwBHhTKozhM3oEQmvFvckGuVLDY6dzsPVHoE1uTXjGk |
| project_id | f04ec0abf3d1460dad82608bb03af589                                                                                                                                                        |
| user_id    | 8533cb3873974fa29f03832aef7007ca                                                                                                                                                        |
+------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
root@controller:~# 
```

keystone 服务正常

那这时，看一下 keystone 日志，应该会能接收到请求。

但是，实际观察中，keystone 日志，没有收到请求。这说明，swift 本身的问题，没有发出请求到 keystone 。

### swift 访问是否正常

```
root@controller:~# swift --os-auth-url http://controller:5000/v3 --auth-version 3\
>         --os-project-name demo --os-project-domain-name Default \
>         --os-username demo --os-user-domain-name Default \
>         --os-password openstack list

Auth versions 2.0 and 3 require python-keystoneclient, install it or use Auth
version 1.0 which requires ST_AUTH, ST_USER, and ST_KEY environment
variables to be set or overridden with -A, -U, or -K.
root@controller:~# 
```


看一下版本与位置

```
root@controller:~# swift --version
python-swiftclient 3.6.0
root@controller:~# which swift
/usr/local/bin/swift
root@controller:~# 
```

但是，观察了另一台机器的swift 却是在 /usr/bin/swift 下的。
版本不一样。

```
root@controller:~# /usr/bin/swift --version
python-swiftclient 3.5.0
root@controller:~# /usr/bin/swift --os-auth-url http://controller:5000/v3 --auth-version 3        --os-project-name demo --os-project-domain-name Default         --os-username demo --os-user-domain-name Default         --os-password openstack list
222
A-DOLcontainer
A-DOLcontainer_segments
LargeFileContainer
LargeFileContainer_segments
container1
container1_segments
container2
jinweilai-work
pub1
test3
test4
testcontainer1
root@controller:~# 
```

原来是swift客户端版本不同导致的。那么使用能正常访问keystone的客户端吧。


```
root@controller:~# mv /usr/local/bin/swift /usr/local/bin/swift.bak
root@controller:~# which  swift
/usr/bin/swift
root@controller:~# . admin-openrc 
root@controller:~# swift list
bash: /usr/local/bin/swift: No such file or directory
root@controller:~# 
```

这样还不行，看一下 PATH 顺序

```
root@controller:~# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:$JAVA_HOME/bin
root@controller:~# /usr/bin/swift stat -v
                     StorageURL: http://192.168.0.50:8080/v1/AUTH_f04ec0abf3d1460dad82608bb03af589
                     Auth Token: gAAAAABcf22hOyYW2Qfa9JS7rKMiNScbVhEY-UtqJ_DsL86x5hXP_ZvMPCp5qHicxrMT3hJSWMPPXexYm1MkDzx43fvqbfXKr3UpbFQFTto4zqb51Rhc61iHyC4LQi5c2oK9VVU_5rLSkg0mNR6OALLvhOWzAgZHqQrZzMpLJ0SGwYrBtfgO-3Y
                        Account: AUTH_f04ec0abf3d1460dad82608bb03af589
                     Containers: 2
                        Objects: 2
                          Bytes: 26264931
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 2
     Bytes in policy "policy-0": 26264931
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txe47025f385c84cc4a6821-005c7f6da1
                    X-Timestamp: 1547641627.86782
                     X-Trans-Id: txe47025f385c84cc4a6821-005c7f6da1
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# /usr/bin/swift list
DOLcontainer
container1
root@controller:~# which swift
/usr/bin/swift
root@controller:~# swift list
bash: /usr/local/bin/swift: No such file or directory
```

又还原了。那就直接建立 软链接 过去吧。

```
root@controller:~# ln -s /usr/bin/swift /usr/local/bin/swift
root@controller:~# which swift
/usr/local/bin/swift
root@controller:~# swift --version
python-swiftclient 3.5.0
root@controller:~# swift stat -v
                     StorageURL: http://192.168.0.50:8080/v1/AUTH_f04ec0abf3d1460dad82608bb03af589
                     Auth Token: gAAAAABcf23XSjMmOIEeHmAw9XehvC-a4AUwhHKkz7HlKcR0uk1Dm_BMG3ZsiDw_1EMd677DHU4AeXQNmsS7fIkQv7wJSbo8IznVzZaZVgDczNJ-QgfUewvGyOYKs46QwYlitOxFX21l-55Uip58zT3GOvEDZL6TU8Wd2suSeQ6INMHQ2d5kYwA
                        Account: AUTH_f04ec0abf3d1460dad82608bb03af589
                     Containers: 2
                        Objects: 2
                          Bytes: 26264931
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 2
     Bytes in policy "policy-0": 26264931
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txe8d22a79b22b4587be827-005c7f6dd7
                    X-Timestamp: 1547641627.86782
                     X-Trans-Id: txe8d22a79b22b4587be827-005c7f6dd7
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```

现在正常了。

### (忽略)swift.bak 的保留在这里 

```
root@controller:~# /usr/local/bin/swift.bak --version
python-swiftclient 3.6.0
root@controller:~# /usr/local/bin/swift.bak stat -v

Auth versions 2.0 and 3 require python-keystoneclient, install it or use Auth
version 1.0 which requires ST_AUTH, ST_USER, and ST_KEY environment
variables to be set or overridden with -A, -U, or -K.
root@controller:~# 
```



