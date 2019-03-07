
一次 admin 用户在horizon无法登录的事故

## env

admin 用户在horizon无法登录
controller 节点上：
- swift stat 无效
- openstack endpoint list 无效
- openstack 其它操作 无效

后来，才知道：admin用户 被删除了（跑路么...嘿嘿）

## step

### 查数据库是否正常

```
root@ubuntu:/home/administrator# systemctl status mariadb
● mariadb.service - MariaDB 10.2.22 database server
   Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: active (running) since 一 2019-03-04 17:39:31 CST; 17h ago
     Docs: man:mysqld(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 2056 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 2052 ExecStartPost=/etc/mysql/debian-start (code=exited, status=0/SUCCESS)
  Process: 1171 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`/usr/bin/galera_recovery`; [ $? -eq 0 ]   && systemctl set-environment _WSREP_START_POSITION=$VAR || exit 1 (code=exited, status=0/SUCCESS)
  Process: 1139 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 1105 ExecStartPre=/usr/bin/install -m 755 -o mysql -g root -d /var/run/mysqld (code=exited, status=0/SUCCESS)
 Main PID: 1707 (mysqld)
   Status: "Taking your SQL requests now..."
    Tasks: 36
   Memory: 104.8M
      CPU: 43.268s
   CGroup: /system.slice/mariadb.service
           └─1707 /usr/sbin/mysqld

3月 05 10:36:56 ubuntu mysqld[1707]: 2019-03-05 10:36:56 139938802427648 [Warning] Aborted connection 33 to db: 'keystone' user: 'keystone' host: 'controller' (Got timeout reading communication packets)
3月 05 10:43:11 ubuntu mysqld[1707]: 2019-03-05 10:43:11 139939162048256 [Warning] Aborted connection 34 to db: 'keystone' user: 'keystone' host: 'controller' (Got timeout reading communication packets)
3月 05 10:43:11 ubuntu mysqld[1707]: 2019-03-05 10:43:11 139938802124544 [Warning] Aborted connection 37 to db: 'keystone' user: 'keystone' host: 'controller' (Got timeout reading communication packets)
3月 05 10:43:11 ubuntu mysqld[1707]: 2019-03-05 10:43:11 139939161745152 [Warning] Aborted connection 35 to db: 'keystone' user: 'keystone' host: 'controller' (Got timeout reading communication packets)
3月 05 11:00:18 ubuntu mysqld[1707]: 2019-03-05 11:00:18 139939162048256 [Warning] Aborted connection 40 to db: 'keystone' user: 'keystone' host: 'controller' (Got timeout reading communication packets)
3月 05 11:08:11 ubuntu mysqld[1707]: 2019-03-05 11:08:11 139939161745152 [Warning] Aborted connection 38 to db: 'keystone' user: 'keystone' host: 'controller' (Got an error reading communication packets)
3月 05 11:08:11 ubuntu mysqld[1707]: 2019-03-05 11:08:11 139939162351360 [Warning] Aborted connection 42 to db: 'keystone' user: 'keystone' host: 'controller' (Got an error reading communication packets)
3月 05 11:08:11 ubuntu mysqld[1707]: 2019-03-05 11:08:11 139938802427648 [Warning] Aborted connection 41 to db: 'keystone' user: 'keystone' host: 'controller' (Got an error reading communication packets)
3月 05 11:08:11 ubuntu mysqld[1707]: 2019-03-05 11:08:11 139938802124544 [Warning] Aborted connection 39 to db: 'keystone' user: 'keystone' host: 'controller' (Got an error reading communication packets)
3月 05 11:08:11 ubuntu mysqld[1707]: 2019-03-05 11:08:11 139939162048256 [Warning] Aborted connection 43 to db: 'keystone' user: 'keystone' host: 'controller' (Got an error reading communication packets)
root@ubuntu:/home/administrator# netstat -tnlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      1902/beam       
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      1707/mysqld     
tcp        0      0 192.168.10.13:2379      0.0.0.0:*               LISTEN      1115/etcd       
tcp        0      0 0.0.0.0:11211           0.0.0.0:*               LISTEN      1111/memcached  
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      2073/python     
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      1521/epmd       
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1504/sshd       
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      17959/2         
tcp6       0      0 :::5000                 :::*                    LISTEN      18114/apache2   
tcp6       0      0 :::5672                 :::*                    LISTEN      1902/beam       
tcp6       0      0 :::2380                 :::*                    LISTEN      1115/etcd       
tcp6       0      0 :::80                   :::*                    LISTEN      18114/apache2   
tcp6       0      0 :::4369                 :::*                    LISTEN      1521/epmd       
tcp6       0      0 :::22                   :::*                    LISTEN      1504/sshd       
tcp6       0      0 ::1:6010                :::*                    LISTEN      17959/2         
root@ubuntu:/home/administrator# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 50
Server version: 10.2.22-MariaDB-10.2.22+maria~xenial-log mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
root@ubuntu:/home/administrator# mysql -u keystone -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 51
Server version: 10.2.22-MariaDB-10.2.22+maria~xenial-log mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

keystone 用户是正常的呀...

这时，你怎么查，也查不出问题的。。。

后来，同事反馈，因为，测试工程师，不小心，把admin用户给删除了！~ 太尴尬了。

```
root@ubuntu:/home/administrator# . admin-openrc 
root@ubuntu:/home/administrator# openstack endpoint list
The request you have made requires authentication. (HTTP 401) (Request-ID: req-9107092c-98fd-4462-89a1-bd0ce4900e39)
root@ubuntu:/home/administrator# 
```
### keystone 加 admin 用户

参考 https://docs.openstack.org/keystone/rocky/install/keystone-install-ubuntu.html#install-and-configure-components 第5小步 Bootstrap the Identity service ：


```
root@ubuntu:/home/administrator# keystone-manage bootstrap --bootstrap-password 962ae831854bbd768a2f \
>   --bootstrap-admin-url http://controller:5000/v3/ \
>   --bootstrap-internal-url http://controller:5000/v3/ \
>   --bootstrap-public-url http://controller:5000/v3/ \
>   --bootstrap-region-id RegionOne
root@ubuntu:/home/administrator# service apache2 restart
```

### 验证

```
root@ubuntu:/home/administrator# . admin-openrc 
root@ubuntu:/home/administrator# openstack --os-auth-url http://controller:5000/v3 \
>   --os-project-domain-name Default --os-user-domain-name Default \
>   --os-project-name admin --os-username admin token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-03-05T05:11:19+0000                                                                                                                                                                |
| id         | gAAAAABcffbnfJIQuwWelSU7NkenAkXopEbJW6fqPE2mYPC0z95gMlAptMUsnbVe1UhA4HVnollox_TSYaQHJ8EgFi4SFWeQlJT9H_gVhpih-g_YQY7yetb5HolBv-S_ludzvW6Tr2GSdc3S4fpEZiYJZ3FqGQEBXfnK5VX7fG28xoVw1EsVwL0 |
| project_id | 544bf3a68de64987ae3bd92f640facc4                                                                                                                                                        |
| user_id    | 89bd983df24742ff8892d190a06d5f58                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
root@ubuntu:/home/administrator# 
```

admin用户还原成功了！~ 

### swift stat OK


```
root@ubuntu:/home/administrator# . admin-openrc 
root@ubuntu:/home/administrator# swift stat -v
Account HEAD failed: http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4 503 Service Unavailable
Failed Transaction ID: tx3c28aa4aa7264929a015d-005c7dfba0
root@ubuntu:/home/administrator# openstack user list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 1319e9203c0c4000819b625bf987742c | swift |
| 89bd983df24742ff8892d190a06d5f58 | admin |
| c3248a1ddb4048349796e4e523d03b54 | demo  |
+----------------------------------+-------+
root@ubuntu:/home/administrator# openstack role add --project service --user swift admin
root@ubuntu:/home/administrator# openstack role list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| a9c6feb32e8d4d099e0140111c9fc137 | admin |
| dfbf44be3cfe4f4790944bfc4300a3ba | user  |
+----------------------------------+-------+
root@ubuntu:/home/administrator# openstack service create --name swift \
>   --description "OpenStack Object Storage" object-store
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Object Storage         |
| enabled     | True                             |
| id          | 31a05ac2789644b29cea4c561e094f3a |
| name        | swift                            |
| type        | object-store                     |
+-------------+----------------------------------+
root@ubuntu:/home/administrator# openstack endpoint create --region RegionOne \
>   object-store public http://controller:8080/v1/AUTH_%\(project_id\)s
Multiple service matches found for 'object-store', use an ID to be more specific.
root@ubuntu:/home/administrator# openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                           |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------------------+
| 3bd2e6635a1e469ab5284c11d859e4b3 | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3/                    |
| 3fc42de3790b4c5f8d635a9721ed23a6 | RegionOne | swift        | object-store | True    | public    | http://controller:8080/v1/AUTH_%(project_id)s |
| 88634b7b1e6c4b0989f079f00adbffb8 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3/                    |
| 8b4faa17e4404ef4aa709c0e78636742 | RegionOne | swift        | object-store | True    | internal  | http://controller:8080/v1/AUTH_%(project_id)s |
| bf0706eea0e04412ae4588ec9dd067a0 | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3/                    |
| bfae1eb954c84364803023604999ef93 | RegionOne | swift        | object-store | True    | admin     | http://controller:8080/v1                     |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------------------+
root@ubuntu:/home/administrator# openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                           |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------------------+
| 3bd2e6635a1e469ab5284c11d859e4b3 | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3/                    |
| 3fc42de3790b4c5f8d635a9721ed23a6 | RegionOne | swift        | object-store | True    | public    | http://controller:8080/v1/AUTH_%(project_id)s |
| 88634b7b1e6c4b0989f079f00adbffb8 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3/                    |
| 8b4faa17e4404ef4aa709c0e78636742 | RegionOne | swift        | object-store | True    | internal  | http://controller:8080/v1/AUTH_%(project_id)s |
| bf0706eea0e04412ae4588ec9dd067a0 | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3/                    |
| bfae1eb954c84364803023604999ef93 | RegionOne | swift        | object-store | True    | admin     | http://controller:8080/v1                     |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------------------+
root@ubuntu:/home/administrator# openstack endpoint create --region RegionOne \
>   object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
Multiple service matches found for 'object-store', use an ID to be more specific.
root@ubuntu:/home/administrator# . admin-openrc 
root@ubuntu:/home/administrator# openstack endpoint create --region RegionOne   object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
Multiple service matches found for 'object-store', use an ID to be more specific.
root@ubuntu:/home/administrator# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since 一 2019-03-04 17:39:48 CST; 18h ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1264 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 2
   Memory: 124.6M
      CPU: 6min 37.175s
   CGroup: /system.slice/swift-proxy.service
           ├─2073 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─2084 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

3月 05 12:31:11 ubuntu proxy-server[2084]: Identity response: {"error": {"message": "The request you have made requires authentication.", "code": 401, "title": "Unauthorized"}}
3月 05 12:31:11 ubuntu proxy-server[2084]: Unable to validate token: Identity server rejected authorization necessary to fetch token data
3月 05 12:31:12 ubuntu proxy-server[2084]: 192.168.10.13 192.168.10.13 05/Mar/2019/04/31/12 HEAD /v1/AUTH_544bf3a68de64987ae3bd92f640facc4%3Fformat%3Djson HTTP/1.0 503 - python-swift
3月 05 12:31:28 ubuntu proxy-server[2084]: Identity server rejected authorization
3月 05 12:31:28 ubuntu proxy-server[2084]: Identity response: {"error": {"message": "The request you have made requires authentication.", "code": 401, "title": "Unauthorized"}}
3月 05 12:31:28 ubuntu proxy-server[2084]: Retrying validation
3月 05 12:31:29 ubuntu proxy-server[2084]: Identity server rejected authorization
3月 05 12:31:29 ubuntu proxy-server[2084]: Identity response: {"error": {"message": "The request you have made requires authentication.", "code": 401, "title": "Unauthorized"}}
3月 05 12:31:29 ubuntu proxy-server[2084]: Unable to validate token: Identity server rejected authorization necessary to fetch token data
3月 05 12:31:29 ubuntu proxy-server[2084]: 192.168.10.13 192.168.10.13 05/Mar/2019/04/31/29 HEAD /v1/AUTH_544bf3a68de64987ae3bd92f640facc4%3Fformat%3Djson HTTP/1.0 503 - python-swift
root@ubuntu:/home/administrator# service swift-proxy restart
root@ubuntu:/home/administrator# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since 二 2019-03-05 12:36:31 CST; 28s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 20857 ExecStop=/etc/init.d/swift-proxy stop (code=exited, status=0/SUCCESS)
  Process: 20871 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 2
   Memory: 88.1M
      CPU: 3.284s
   CGroup: /system.slice/swift-proxy.service
           ├─20882 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─20891 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

3月 05 12:36:28 ubuntu proxy-server[20882]: Started child 20891
3月 05 12:36:28 ubuntu proxy-server[20891]: Adding required filter copy to pipeline at position 10
3月 05 12:36:28 ubuntu proxy-server[20891]: Adding required filter listing_formats to pipeline at position 5
3月 05 12:36:28 ubuntu proxy-server[20891]: Pipeline was modified. New pipeline is "catch_errors gatekeeper healthcheck proxy-logging cache listing_formats container_sync bulk rateli
3月 05 12:36:28 ubuntu proxy-server[20891]: Starting Keystone auth_token middleware
3月 05 12:36:28 ubuntu proxy-server[20891]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated
3月 05 12:36:31 ubuntu swift-proxy[20871]: Starting proxy-server...(/etc/swift/proxy-server.conf)
3月 05 12:36:31 ubuntu swift-proxy[20871]: No handlers could be found for logger "keystonemiddleware._common.config"
3月 05 12:36:31 ubuntu swift-proxy[20871]:    ...done.
3月 05 12:36:31 ubuntu systemd[1]: Started LSB: Swift proxy server.
root@ubuntu:/home/administrator# . admin-openrc 
root@ubuntu:/home/administrator# swift stat -v
                     StorageURL: http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4
                     Auth Token: gAAAAABcffz0ZaBYiClGBQWWG80xSRlldlvyAqboD9x6uPgn38DgnZICHzeDCLF7_EJUVX4Uljn_IzQucX6502N85_ZvLQLCs-OFbUrRKu5H7fqMrLrXyNZnZlanXNglghb9VRa8BRUGddxnms2y4EWofDj2XxunYQVuHzWlym37ghCbw8QCrRY
                        Account: AUTH_544bf3a68de64987ae3bd92f640facc4
                     Containers: 3
                        Objects: 7
                          Bytes: 1019964752
Containers in policy "policy-0": 3
   Objects in policy "policy-0": 7
     Bytes in policy "policy-0": 1019964752
    X-Account-Project-Domain-Id: default
         X-Openstack-Request-Id: txed77851303b14b9f934b5-005c7dfcf4
                    X-Timestamp: 1550483325.13780
                     X-Trans-Id: txed77851303b14b9f934b5-005c7dfcf4
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@ubuntu:/home/administrator# swift list
container1
test
testtest
root@ubuntu:/home/administrator# 
```

现在 swift 已经可以连接上了。

###  horizon 登录后， `无法获取swift服务信息`，重启 memcached 

但是 通过 horizon 连接登录后，还是会显示 `无法获取swift服务信息` 。

因为安装是依据 https://docs.openstack.org/horizon/queens/install/install-ubuntu.html 

配置在 `/etc/openstack-dashboard/local_settings.py`, 无修改。

日志在 apache2 上，看一下：

```
administrator@ubuntu:~$ tail -f /var/log/apache2/access.log
192.168.10.15 - - [05/Mar/2019:14:34:17 +0800] "GET /horizon/project/containers/ HTTP/1.1" 200 4781 "-" "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"
192.168.10.15 - - [05/Mar/2019:14:34:17 +0800] "GET /horizon/i18n/js/horizon+openstack_dashboard/ HTTP/1.1" 200 108038 "http://192.168.10.13/horizon/project/containers/" "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"
192.168.10.15 - - [05/Mar/2019:14:34:17 +0800] "GET /horizon/api/swift/info/ HTTP/1.1" 500 320 "http://192.168.10.13/horizon/project/containers/" "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"
192.168.10.15 - - [05/Mar/2019:14:34:17 +0800] "GET /horizon/api/swift/containers/ HTTP/1.1" 500 320 "http://192.168.10.13/horizon/project/containers/" "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"
192.168.10.15 - - [05/Mar/2019:14:34:17 +0800] "GET /horizon/api/swift/containers/ HTTP/1.1" 500 320 "http://192.168.10.13/horizon/project/containers/" "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"
192.168.10.15 - - [05/Mar/2019:14:34:17 +0800] "GET /horizon/header/ HTTP/1.1" 200 416 "http://192.168.10.13/horizon/project/containers/" "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36"
```

GET /horizon/api/swift/info/ HTTP/1.1" 500

看来 swift 返回不对。看一下apache2, keystone和swift 日志。

- /var/log/keystone/keystone-wsgi-public.log

```
root@ubuntu:/var/log/keystone# tail -f /var/log/keystone/keystone-wsgi-public.log
2019-03-05 15:41:33.335 23839 INFO keystone.common.wsgi [req-abe7bad4-5f1c-4981-beea-008cd27d0023 89bd983df24742ff8892d190a06d5f58 - - default -] GET http://controller:5000/v3/users/89bd983df24742ff8892d190a06d5f58/projects
2019-03-05 15:41:34.056 23835 INFO keystone.common.wsgi [req-d61ed24d-6934-471d-bdfd-084e03823608 89bd983df24742ff8892d190a06d5f58 - - default -] GET http://controller:5000/v3/users/89bd983df24742ff8892d190a06d5f58/projects
```

- swift 日志 没反应，应该是还没有到这里来。

- apache error log： `/var/log/apache2/error.log`

```
root@ubuntu:/var/log/keystone# tail -f /var/log/apache2/error.log
[Tue Mar 05 08:48:48.446270 2019] [wsgi:error] [pid 26080:tid 140389557745408] ERROR openstack_dashboard.api.rest.utils HTTP exception with no status/code
[Tue Mar 05 08:48:48.446902 2019] [wsgi:error] [pid 26078:tid 140389482137344] ERROR openstack_dashboard.api.rest.utils HTTP exception with no status/code
[Tue Mar 05 08:48:48.447043 2019] [wsgi:error] [pid 26078:tid 140389482137344] Traceback (most recent call last):
[Tue Mar 05 08:48:48.447159 2019] [wsgi:error] [pid 26078:tid 140389482137344]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/rest/utils.py", line 127, in _wrapped
[Tue Mar 05 08:48:48.447259 2019] [wsgi:error] [pid 26078:tid 140389482137344]     data = function(self, request, *args, **kw)
[Tue Mar 05 08:48:48.447369 2019] [wsgi:error] [pid 26078:tid 140389482137344]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/rest/swift.py", line 40, in get
[Tue Mar 05 08:48:48.447473 2019] [wsgi:error] [pid 26078:tid 140389482137344]     capabilities = api.swift.swift_get_capabilities(request)
[Tue Mar 05 08:48:48.447583 2019] [wsgi:error] [pid 26078:tid 140389482137344]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/swift.py", line 382, in swift_get_capabilities
[Tue Mar 05 08:48:48.447680 2019] [wsgi:error] [pid 26078:tid 140389482137344]     return swift_api(request).get_capabilities()
[Tue Mar 05 08:48:48.447788 2019] [wsgi:error] [pid 26078:tid 140389482137344]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/swift.py", line 107, in swift_api
[Tue Mar 05 08:48:48.447896 2019] [wsgi:error] [pid 26078:tid 140389482137344]     endpoint = base.url_for(request, 'object-store')
[Tue Mar 05 08:48:48.448004 2019] [wsgi:error] [pid 26080:tid 140389557745408] Traceback (most recent call last):
[Tue Mar 05 08:48:48.448100 2019] [wsgi:error] [pid 26078:tid 140389482137344]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/base.py", line 341, in url_for
[Tue Mar 05 08:48:48.448212 2019] [wsgi:error] [pid 26080:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/rest/utils.py", line 127, in _wrapped
[Tue Mar 05 08:48:48.448309 2019] [wsgi:error] [pid 26078:tid 140389482137344]     raise exceptions.ServiceCatalogException(service_type)
[Tue Mar 05 08:48:48.448418 2019] [wsgi:error] [pid 26080:tid 140389557745408]     data = function(self, request, *args, **kw)
[Tue Mar 05 08:48:48.448513 2019] [wsgi:error] [pid 26078:tid 140389482137344] ServiceCatalogException: Invalid service catalog: object-store
[Tue Mar 05 08:48:48.448623 2019] [wsgi:error] [pid 26080:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/rest/swift.py", line 63, in get
[Tue Mar 05 08:48:48.449208 2019] [wsgi:error] [pid 26080:tid 140389557745408]     containers, has_more = api.swift.swift_get_containers(request)
[Tue Mar 05 08:48:48.449310 2019] [wsgi:error] [pid 26080:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/swift.py", line 141, in swift_get_containers
[Tue Mar 05 08:48:48.449409 2019] [wsgi:error] [pid 26080:tid 140389557745408]     headers, containers = swift_api(request).get_account(limit=limit + 1,
[Tue Mar 05 08:48:48.449518 2019] [wsgi:error] [pid 26080:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/swift.py", line 107, in swift_api
[Tue Mar 05 08:48:48.449616 2019] [wsgi:error] [pid 26080:tid 140389557745408]     endpoint = base.url_for(request, 'object-store')
[Tue Mar 05 08:48:48.449723 2019] [wsgi:error] [pid 26080:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/base.py", line 341, in url_for
[Tue Mar 05 08:48:48.449820 2019] [wsgi:error] [pid 26080:tid 140389557745408]     raise exceptions.ServiceCatalogException(service_type)
[Tue Mar 05 08:48:48.449931 2019] [wsgi:error] [pid 26080:tid 140389557745408] ServiceCatalogException: Invalid service catalog: object-store
[Tue Mar 05 08:48:48.474587 2019] [wsgi:error] [pid 26078:tid 140389557745408] ERROR openstack_dashboard.api.rest.utils HTTP exception with no status/code
[Tue Mar 05 08:48:48.474819 2019] [wsgi:error] [pid 26078:tid 140389557745408] Traceback (most recent call last):
[Tue Mar 05 08:48:48.474925 2019] [wsgi:error] [pid 26078:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/rest/utils.py", line 127, in _wrapped
[Tue Mar 05 08:48:48.475053 2019] [wsgi:error] [pid 26078:tid 140389557745408]     data = function(self, request, *args, **kw)
[Tue Mar 05 08:48:48.475153 2019] [wsgi:error] [pid 26078:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/rest/swift.py", line 63, in get
[Tue Mar 05 08:48:48.475268 2019] [wsgi:error] [pid 26078:tid 140389557745408]     containers, has_more = api.swift.swift_get_containers(request)
[Tue Mar 05 08:48:48.475378 2019] [wsgi:error] [pid 26078:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/swift.py", line 141, in swift_get_containers
[Tue Mar 05 08:48:48.475490 2019] [wsgi:error] [pid 26078:tid 140389557745408]     headers, containers = swift_api(request).get_account(limit=limit + 1,
[Tue Mar 05 08:48:48.475589 2019] [wsgi:error] [pid 26078:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/swift.py", line 107, in swift_api
[Tue Mar 05 08:48:48.475737 2019] [wsgi:error] [pid 26078:tid 140389557745408]     endpoint = base.url_for(request, 'object-store')
[Tue Mar 05 08:48:48.475834 2019] [wsgi:error] [pid 26078:tid 140389557745408]   File "/usr/share/openstack-dashboard/openstack_dashboard/api/base.py", line 341, in url_for
[Tue Mar 05 08:48:48.475960 2019] [wsgi:error] [pid 26078:tid 140389557745408]     raise exceptions.ServiceCatalogException(service_type)
[Tue Mar 05 08:48:48.476057 2019] [wsgi:error] [pid 26078:tid 140389557745408] ServiceCatalogException: Invalid service catalog: object-store
```

也就是说，chrome => apache2 => horizon => keystone => swift 中的 horizon => keystone 出现了问题，也就是认证出问题了。

根据这个提示，来到 `/usr/share/openstack-dashboard/openstack_dashboard/api/base.py` 的 341 行，这个地方，加一些调试信息

```
def url_for(request, service_type, endpoint_type=None, region=None):
    endpoint_type = endpoint_type or getattr(settings,
                                             'OPENSTACK_ENDPOINT_TYPE',
                                             'publicURL')
    fallback_endpoint_type = getattr(settings, 'SECONDARY_ENDPOINT_TYPE', None)

    catalog = request.user.service_catalog
    service = get_service_from_catalog(catalog, service_type)
    print("endpoint_type:::", endpoint_type)
    print("fallback_endpoint_type:::",  fallback_endpoint_type)
    print(" catalog, service:::",  catalog)
    print(" service:::",  service)
    if service:
        if not region:
            region = request.user.services_region
        url = get_url_for_service(service,
                                  region,
                                  endpoint_type)
        if not url and fallback_endpoint_type:
            url = get_url_for_service(service,
                                      region,
                                      fallback_endpoint_type)
        if url:
            return url
    raise exceptions.ServiceCatalogException(service_type)
```

`service apache2 restart` 重启 apache2 , 输出信息如下：

```
[Tue Mar 05 09:07:26.208745 2019] [wsgi:error] [pid 27029:tid 139698939741952] ('endpoint_type:::', 'publicURL')
[Tue Mar 05 09:07:26.208944 2019] [wsgi:error] [pid 27029:tid 139698939741952] ('fallback_endpoint_type:::', None)
[Tue Mar 05 09:07:26.209129 2019] [wsgi:error] [pid 27029:tid 139698939741952] (' catalog, service:::', [{u'endpoints': [], u'type': u'object-store', u'id': u'31a05ac2789644b29cea4c561e094f3a', u'name': u'swift'}, {u'endpoints': [{u'url': u'http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4', u'interface': u'public', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'3fc42de3790b4c5f8d635a9721ed23a6'}, {u'url': u'http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4', u'interface': u'internal', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'8b4faa17e4404ef4aa709c0e78636742'}, {u'url': u'http://controller:8080/v1', u'interface': u'admin', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'bfae1eb954c84364803023604999ef93'}], u'type': u'object-store', u'id': u'c9c91a6a09f34a129dcf587f5e006e33', u'name': u'swift'}, {u'endpoints': [{u'url': u'http://controller:5000/v3/', u'interface': u'admin', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'093cee60e5a144e3a65441fa2db6511e'}, {u'url': u'http://controller:5000/v3/', u'interface': u'internal', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'88634b7b1e6c4b0989f079f00adbffb8'}, {u'url': u'http://controller:5000/v3/', u'interface': u'public', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'bf0706eea0e04412ae4588ec9dd067a0'}], u'type': u'identity', u'id': u'e16c2dfb748942c98f57b7cbe3c522c7', u'name': u'keystone'}])
[Tue Mar 05 09:07:26.209279 2019] [wsgi:error] [pid 27029:tid 139698939741952] (' service:::', {u'endpoints': [], u'type': u'object-store', u'id': u'31a05ac2789644b29cea4c561e094f3a', u'name': u'swift'})
[Tue Mar 05 09:07:26.210264 2019] [wsgi:error] [pid 27029:tid 139698939741952] 
```

可以看到，catalog 与 service 中的 endpoint 为空。说明有空的。

```
root@ubuntu:/etc/apache2# openstack catalog list
+----------+--------------+-----------------------------------------------------------------------------+
| Name     | Type         | Endpoints                                                                   |
+----------+--------------+-----------------------------------------------------------------------------+
| swift    | object-store |                                                                             |
| swift    | object-store | RegionOne                                                                   |
|          |              |   public: http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4   |
|          |              | RegionOne                                                                   |
|          |              |   internal: http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4 |
|          |              | RegionOne                                                                   |
|          |              |   admin: http://controller:8080/v1                                          |
|          |              |                                                                             |
| keystone | identity     | RegionOne                                                                   |
|          |              |   admin: http://controller:5000/v3/                                         |
|          |              | RegionOne                                                                   |
|          |              |   internal: http://controller:5000/v3/                                      |
|          |              | RegionOne                                                                   |
|          |              |   public: http://controller:5000/v3/                                        |
|          |              |                                                                             |
+----------+--------------+-----------------------------------------------------------------------------+
root@ubuntu:/etc/apache2# openstack service list
+----------------------------------+----------+--------------+
| ID                               | Name     | Type         |
+----------------------------------+----------+--------------+
| 1d812c20c2a34a4f93b9ce317d004de6 | swift    | object-store |
| c9c91a6a09f34a129dcf587f5e006e33 | swift    | object-store |
| e16c2dfb748942c98f57b7cbe3c522c7 | keystone | identity     |
+----------------------------------+----------+--------------+
```

那有可能系统取了第一个的 catalog , 尝试把它删除

```
root@ubuntu:/etc/apache2# openstack catalog show swift
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| endpoints |                                  |
| id        | 1d812c20c2a34a4f93b9ce317d004de6 |
| name      | swift                            |
| type      | object-store                     |
+-----------+----------------------------------+
root@ubuntu:/etc/apache2# openstack catalog delete 1d812c20c2a34a4f93b9ce317d004de6
openstack: 'catalog delete 1d812c20c2a34a4f93b9ce317d004de6' is not an openstack command. See 'openstack --help'.
Did you mean one of these?
  catalog list
  catalog show
  console log show
  console url show
```

注意观察此 id 不是catalog 的，而是 service 的，那就把这个service 删除。

```
root@ubuntu:/etc/apache2# openstack service show 1d812c20c2a34a4f93b9ce317d004de6
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Object Storage2        |
| enabled     | True                             |
| id          | 1d812c20c2a34a4f93b9ce317d004de6 |
| name        | swift                            |
| type        | object-store                     |
+-------------+----------------------------------+
root@ubuntu:/etc/apache2# openstack service delete 1d812c20c2a34a4f93b9ce317d004de6
root@ubuntu:/etc/apache2# openstack catalog list
+----------+--------------+-----------------------------------------------------------------------------+
| Name     | Type         | Endpoints                                                                   |
+----------+--------------+-----------------------------------------------------------------------------+
| swift    | object-store | RegionOne                                                                   |
|          |              |   public: http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4   |
|          |              | RegionOne                                                                   |
|          |              |   internal: http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4 |
|          |              | RegionOne                                                                   |
|          |              |   admin: http://controller:8080/v1                                          |
|          |              |                                                                             |
| keystone | identity     | RegionOne                                                                   |
|          |              |   admin: http://controller:5000/v3/                                         |
|          |              | RegionOne                                                                   |
|          |              |   internal: http://controller:5000/v3/                                      |
|          |              | RegionOne                                                                   |
|          |              |   public: http://controller:5000/v3/                                        |
|          |              |                                                                             |
+----------+--------------+-----------------------------------------------------------------------------+
root@ubuntu:/etc/apache2# 
```

`service apache2 restart` 重启 apache2 , 输出信息如下：

```
[Tue Mar 05 09:07:26.208745 2019] [wsgi:error] [pid 27029:tid 139698939741952] ('endpoint_type:::', 'publicURL')
[Tue Mar 05 09:07:26.208944 2019] [wsgi:error] [pid 27029:tid 139698939741952] ('fallback_endpoint_type:::', None)
[Tue Mar 05 09:07:26.209129 2019] [wsgi:error] [pid 27029:tid 139698939741952] (' catalog, service:::', [{u'endpoints': [], u'type': u'object-store', u'id': u'31a05ac2789644b29cea4c561e094f3a', u'name': u'swift'}, {u'endpoints': [{u'url': u'http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4', u'interface': u'public', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'3fc42de3790b4c5f8d635a9721ed23a6'}, {u'url': u'http://controller:8080/v1/AUTH_544bf3a68de64987ae3bd92f640facc4', u'interface': u'internal', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'8b4faa17e4404ef4aa709c0e78636742'}, {u'url': u'http://controller:8080/v1', u'interface': u'admin', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'bfae1eb954c84364803023604999ef93'}], u'type': u'object-store', u'id': u'c9c91a6a09f34a129dcf587f5e006e33', u'name': u'swift'}, {u'endpoints': [{u'url': u'http://controller:5000/v3/', u'interface': u'admin', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'093cee60e5a144e3a65441fa2db6511e'}, {u'url': u'http://controller:5000/v3/', u'interface': u'internal', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'88634b7b1e6c4b0989f079f00adbffb8'}, {u'url': u'http://controller:5000/v3/', u'interface': u'public', u'region': u'RegionOne', u'region_id': u'RegionOne', u'id': u'bf0706eea0e04412ae4588ec9dd067a0'}], u'type': u'identity', u'id': u'e16c2dfb748942c98f57b7cbe3c522c7', u'name': u'keystone'}])
[Tue Mar 05 09:07:26.209279 2019] [wsgi:error] [pid 27029:tid 139698939741952] (' service:::', {u'endpoints': [], u'type': u'object-store', u'id': u'31a05ac2789644b29cea4c561e094f3a', u'name': u'swift'})
[Tue Mar 05 09:07:26.210264 2019] [wsgi:error] [pid 27029:tid 139698939741952] 
```

什么情况，居然，没有变化。那明显不对呀。什么问题...缓存...

`service memcached restart`

OK。这下成功了。刷新 chrome ，可以得到正确的结果了。

