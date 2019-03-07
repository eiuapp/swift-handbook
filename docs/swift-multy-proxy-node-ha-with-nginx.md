
Openstack存储swift多代理节点通过nginx实现HA

![swift-multy-proxy-node-architecture](https://res.cloudinary.com/dmtixvmgt/image/upload/v1548240314/swift-multy-proxy-node-architecture_cjcb3m.jpg)

上图 Load Balancer 由 nginx upsteam 来实现

## env

- 系统：Ubuntu Server 16.04×64 
- 存储设置：4T
- 架构部署: 

| 主机名   |     IP      |  作用 |
|----------|:-------------:|------:|
| Proxy |  192.168.0.51 | keystone |
| Proxybak  |        192.168.0.142   |    nginx | 
| Proxy |  192.168.0.51 | 代理节点  |
| Proxybak  |        192.168.0.141   |    代理节点, 做冗余备份| 
| object1      |     192.168.0.127    |     存储节点1(zone1)| 
| object2      |     192.168.0.134    |     存储节点2(zone1)| 
| object3      |     192.168.0.135    |     存储节点3(zone1)| 
| object4      |    192.168.0.180     |     存储节点4(zone1)| 
| object5      |    192.168.0.189     |     存储节点5(zone1)| 

Assumptions:

- KeyStone V3 is setup and Swift is configured. 
- Python openstack client command line interface is available.
- By default, 'admin' user and 'admin' project is created and assigned to 'default' Domain.  We will use this in this exercise, however you can configure different user, project and domain to accomplish ResellerAdmin setup.
- Openrc file is available to work with openstack CLI client.
 

## 注意

无
## step


### 验证各proxy node 本身有效

分别在 proxy node 运行 `. demo-openrc && swift stat` 

### 配置 nginx

把下面配置加入到 nginx 配置文件 /etc/nginx/nginx.conf 中

```
    upstream swiftproxy { 
        server 192.168.0.141:8080; 
        server 192.168.0.51:8080; 
    }
```

新建立一个 swiftproxy.conf 文件，实现HA

```shell
root@ubuntu:/etc/nginx/conf.d# head -13 swiftproxy.conf 
server {
    listen       8080;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        proxy_pass http://swiftproxy; 
    }

root@ubuntu:/etc/nginx/conf.d# 
```

tail 开启日志

```shell
root@ubuntu:~# tail -f /var/log/nginx/access.log 
192.168.0.129 - - [25/Jan/2019:09:36:59 +0800] "GET /requests/status.xml HTTP/1.1" 502 537 "-" "Dalvik/2.1.0 (Linux; U; Android 8.1.0; vivo NEX S Build/OPM1.171019.026)" "-"
192.168.0.129 - - [25/Jan/2019:09:36:59 +0800] "GET /requests/status.xml HTTP/1.1" 502 537 "-" "Dalvik/2.1.0 (Linux; U; Android 8.1.0; vivo NEX S Build/OPM1.171019.026)" "-"
192.168.0.129 - - [25/Jan/2019:09:37:00 +0800] "GET /requests/status.xml HTTP/1.1" 502 537 "-" "Dalvik/2.1.0 (Linux; U; Android 8.1.0; vivo NEX S Build/OPM1.171019.026)" "-"
```

### （可跳过）有意 写错误的 swiftproxy IP 


```shell
root@controller:~# sudo vi /etc/hosts
root@controller:~# . demo-openrc 
root@controller:~# swift stat
HTTPConnectionPool(host='swiftproxy', port=8080): Max retries exceeded with url: /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 (Caused byoute to host',))
root@controller:~# 
```

### 把 swiftproxy 指向 nginx IP

```shell
root@controller:~# sudo vi /etc/hosts
root@controller:~# grep swiftproxy -rn /etc/hosts
18:# swiftproxy
19:192.168.0.142    swiftproxy
root@controller:~# 
```

### 开始吧

在任意，能发送 swift 请求的地方，向 nginx IP 发送 swift-proxy 相关的请求吧。

```shell
root@controller:~# . demo-openrc 
root@controller:~# swift stat
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 2
                        Objects: 30
                          Bytes: 3718416484
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 30
     Bytes in policy "policy-0": 3718416484
         X-Openstack-Request-Id: tx8b15b33e61dd4e83bca33-005c4a6cd1
    X-Account-Project-Domain-Id: default
                         Server: nginx/1.14.2
                     Connection: keep-alive
                    X-Timestamp: 1547188179.49086
                     X-Trans-Id: tx8b15b33e61dd4e83bca33-005c4a6cd1
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# swift  stat -v
                     StorageURL: http://swiftproxy:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Auth Token: gAAAAABcSmz_UrWKnnqf8yuLfRFR80v_m2xh9sLOwEXuxt9L3lPAHdmY7GuUmLDMRbVg42gDsuY20tI67lxWXk-cwU39eguQcnU1
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 2
                        Objects: 30
                          Bytes: 3718416484
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 30
     Bytes in policy "policy-0": 3718416484
         X-Openstack-Request-Id: txf37fddd19e834f35ba1eb-005c4a6cff
    X-Account-Project-Domain-Id: default
                         Server: nginx/1.14.2
                     Connection: keep-alive
                    X-Timestamp: 1547188179.49086
                     X-Trans-Id: txf37fddd19e834f35ba1eb-005c4a6cff
                   Content-Type: application/json; charset=utf-8
                  Accept-Ranges: bytes
root@controller:~# 
```
### 同时查看 nginx 日志

```shell
root@ubuntu:~# tail -f /var/log/nginx/access.log 
192.168.0.129 - - [25/Jan/2019:09:36:59 +0800] "GET /requests/status.xml HTTP/1.1" 502 537 "-" "Dalvik/2.1.0 (Linux; U; Android 8.1.0; vivo NEX S Build/OPM1.171019.026)" "-"
192.168.0.129 - - [25/Jan/2019:09:36:59 +0800] "GET /requests/status.xml HTTP/1.1" 502 537 "-" "Dalvik/2.1.0 (Linux; U; Android 8.1.0; vivo NEX S Build/OPM1.171019.026)" "-"
192.168.0.129 - - [25/Jan/2019:09:37:00 +0800] "GET /requests/status.xml HTTP/1.1" 502 537 "-" "Dalvik/2.1.0 (Linux; U; Android 8.1.0; vivo NEX S Build/OPM1.171019.026)" "-"
192.168.0.51 - - [25/Jan/2019:09:56:34 +0800] "HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.1" 204 0 "-" "python-swiftclient-3.5.0" "-"


192.168.0.51 - - [25/Jan/2019:09:57:19 +0800] "HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.1" 204 0 "-" "python-swiftclient-3.5.0" "-"
```

有正确请求和响应。


结果正常。说明proxybak node 实现高可用。

操作结束。


## ref

- http://nginx.org/
- https://www.cnblogs.com/wzjhoutai/p/6932007.html
