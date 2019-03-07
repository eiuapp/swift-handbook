swift 常见问题

## chrome 访问： http://192.168.0.50/horizon/auth/login/ 返回 Internal Server Error

返回下面信息

```text
Internal Server Error
The server encountered an internal error or misconfiguration and was unable to complete your request.

Please contact the server administrator at webmaster@localhost to inform them of the time this error occurred, and the actions you performed just before this error.

More information about this error may be available in the server error log.

Apache/2.4.18 (Ubuntu) Server at 192.168.0.50 Port 80
```

然后，我想去修改一下 /etc/hosts 文件

```shell
E514: write error (file system full?)
```

写入错误，磁盘满了？

```shell
root@controller:/etc# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           797M   49M  748M   7% /run
/dev/sda1        19G   19G     0 100% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
tmpfs           797M     0  797M   0% /run/user/1000
root@controller:/etc# 
```

是满了。

## chrome 访问：http://192.168.0.50/horizon/project/containers/ 出现，container 没有展示出来的现象。

然后，我们检查一下 swift 服务

```shell
root@controller:~# . demo-openrc 
root@controller:~# swift list
Account GET failed: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json 503 Service Unavailable  [first 60 chars of response] <html><h1>Service Unavailable</h1><p>The server is currently
Failed Transaction ID: tx1ecd29acbd4e4c889d19d-005c467241
root@controller:~# 
```

报错了。说明 swift 服务挂了。

去 controller 查一下 swift-proxy 服务

```
root@controller:/home/ubuntu# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Tue 2019-01-22 09:18:24 CST; 3min 34s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1597 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 102.7M
      CPU: 3.650s
   CGroup: /system.slice/swift-proxy.service
           ├─2250 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─2556 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─2557 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─2558 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─2559 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Jan 22 09:21:50 controller proxy-server[2559]: Account GET returning 503 for [] (txn: tx8149ac2f570a43589cc19-005c46702e) (client_ip: 127.0.0.1)
Jan 22 09:21:50 controller proxy-server[2559]: 127.0.0.1 127.0.0.1 22/Jan/2019/01/21/50 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Flimit%3D1001%26format%3Djson HTTP/1.0 503 - python-swiftclie
Jan 22 09:21:54 controller proxy-server[2559]: Account HEAD returning 503 for [] (txn: txa6194df8967840519fa19-005c467032)
Jan 22 09:21:54 controller proxy-server[2559]: - - 22/Jan/2019/01/21/54 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 503 - Swift - - - - txa6194df8967840519fa19-005c4670
Jan 22 09:21:54 controller proxy-server[2559]: Account HEAD returning 503 for [] (txn: tx82330a029ccc4710a669f-005c467032)
Jan 22 09:21:54 controller proxy-server[2559]: - - 22/Jan/2019/01/21/54 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 503 - Swift - - - - tx82330a029ccc4710a669f-005c4670
Jan 22 09:21:54 controller proxy-server[2559]: Account GET returning 503 for [] (txn: txa6194df8967840519fa19-005c467032) (client_ip: 127.0.0.1)
Jan 22 09:21:54 controller proxy-server[2559]: 127.0.0.1 127.0.0.1 22/Jan/2019/01/21/54 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Flimit%3D1001%26format%3Djson HTTP/1.0 503 - python-swiftclie
Jan 22 09:21:54 controller proxy-server[2559]: Account GET returning 503 for [] (txn: tx82330a029ccc4710a669f-005c467032) (client_ip: 127.0.0.1)
Jan 22 09:21:54 controller proxy-server[2559]: 127.0.0.1 127.0.0.1 22/Jan/2019/01/21/54 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Flimit%3D1001%26format%3Djson HTTP/1.0 503 - python-swiftclie
root@controller:~# service swift-proxy restart
root@controller:~# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Tue 2019-01-22 09:26:56 CST; 6s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 4688 ExecStop=/etc/init.d/swift-proxy stop (code=exited, status=0/SUCCESS)
  Process: 4699 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 84.2M
      CPU: 945ms
   CGroup: /system.slice/swift-proxy.service
           ├─4711 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─4720 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─4721 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─4722 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─4723 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Jan 22 09:26:53 controller proxy-server[4722]: Starting Keystone auth_token middleware
Jan 22 09:26:53 controller proxy-server[4722]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour.
Jan 22 09:26:53 controller proxy-server[4723]: Starting Keystone auth_token middleware
Jan 22 09:26:53 controller proxy-server[4721]: Starting Keystone auth_token middleware
Jan 22 09:26:53 controller proxy-server[4723]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour.
Jan 22 09:26:53 controller proxy-server[4721]: AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour.
Jan 22 09:26:56 controller swift-proxy[4699]: Starting proxy-server...(/etc/swift/proxy-server.conf)
Jan 22 09:26:56 controller swift-proxy[4699]: No handlers could be found for logger "keystonemiddleware._common.config"
Jan 22 09:26:56 controller swift-proxy[4699]:    ...done.
Jan 22 09:26:56 controller systemd[1]: Started LSB: Swift proxy server.
root@controller:~# . demo-openrc 
root@controller:~# swift list
Account GET failed: http://controller:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json 503 Service Unavailable  [first 60 chars of response] <html><h1>Service Unavailable</h1><p>The server is currently
Failed Transaction ID: tx1ecd29acbd4e4c889d19d-005c467241
root@controller:~# 
```
重启后，依旧，则要去看一下 storage node 中的 `swift-init all status` 

发现确实是 storage node 的问题，则 `swift-init all restart` 。

回到 controller node 检查 `swift list` 就有了，同时，chrome 也有了。



## 当 swift-proxy status 报错时，应该怎么查问题

### proxy node

#### 找问题点

`telnet storageIP 6200` 

如果 telnet 不了，就是对方服务问题，不是 proxy 问题。

### storage node

#### 找问题点

`telnet storageIP 6200`

如果 telnet 不了，就是 服务 问题

#### 服务起来了没有

`netstat -tlnp | grep 6200`

如果 没有，则服务可能没有启动或启动失败。

#### swift-init all 检查

```shell
swift-init all status 
swift-init all restart
```






