
controller node 更换 IP 


```
root@controller:~# . demo-openrc 
root@controller:~# swift stat
^C Aborted
root@controller:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	controller

# controller
192.168.100.50       controller

# compute1
192.168.100.61       compute1

# block1
192.168.100.101       block1

# controller
#192.168.0.51	controller

# storage 
192.168.0.198	swift0198
192.168.0.180	swift0180
192.168.0.135	swift0135

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
root@controller:~# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 11:16:57 CST; 2min 38s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 5510 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 88.5M
      CPU: 2.874s
   CGroup: /system.slice/swift-proxy.service
           ├─5520 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─5529 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─5530 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─5531 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─5532 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Jan 10 11:18:07 controller proxy-server[5530]: STDERR: ERROR:root:Error limiting server controller:11211 (txn: tx6d251040de4d487c8441b-005c
Jan 10 11:18:07 controller proxy-server[5530]: - - 10/Jan/2019/03/18/07 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP
Jan 10 11:18:08 controller keystoneauth.identity.generic.base[5530]: Failed to discover available identity versions when contacting http://
Jan 10 11:18:08 controller proxy-server[5530]: 127.0.0.1 127.0.0.1 10/Jan/2019/03/18/08 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Ffo
Jan 10 11:18:08 controller proxy-server[5530]: Error: An error occurred: #012Traceback (most recent call last):#012  File "/usr/lib/python2
Jan 10 11:18:16 controller proxy-server[5530]: Account HEAD returning 503 for [] (txn: tx95a396094b87489bb2393-005c36b978)
Jan 10 11:18:16 controller proxy-server[5530]: - - 10/Jan/2019/03/18/16 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP
Jan 10 11:18:17 controller keystoneauth.identity.generic.base[5530]: Failed to discover available identity versions when contacting http://
Jan 10 11:18:17 controller proxy-server[5530]: 127.0.0.1 127.0.0.1 10/Jan/2019/03/18/17 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Ffo
Jan 10 11:18:17 controller proxy-server[5530]: Error: An error occurred: #012Traceback (most recent call last):#012  File "/usr/lib/python2
root@controller:~# netstat -nlpt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8774            0.0.0.0:*               LISTEN      2711/python2    
tcp        0      0 0.0.0.0:8775            0.0.0.0:*               LISTEN      2711/python2    
tcp        0      0 0.0.0.0:9191            0.0.0.0:*               LISTEN      2686/python2    
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      2089/beam.smp   
tcp        0      0 192.168.100.50:3306     0.0.0.0:*               LISTEN      2064/mysqld     
tcp        0      0 192.168.100.50:11211    0.0.0.0:*               LISTEN      5494/memcached  
tcp        0      0 192.168.100.50:2379     0.0.0.0:*               LISTEN      1396/etcd       
tcp        0      0 0.0.0.0:9292            0.0.0.0:*               LISTEN      2698/python2    
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      5520/python     
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      1828/epmd       
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1619/sshd       
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      2518/0          
tcp        0      0 0.0.0.0:9696            0.0.0.0:*               LISTEN      2691/python2    
tcp        0      0 0.0.0.0:6080            0.0.0.0:*               LISTEN      2703/python2    
tcp6       0      0 :::5672                 :::*                    LISTEN      2089/beam.smp   
tcp6       0      0 :::5000                 :::*                    LISTEN      2221/apache2    
tcp6       0      0 :::8776                 :::*                    LISTEN      2221/apache2    
tcp6       0      0 :::8778                 :::*                    LISTEN      2221/apache2    
tcp6       0      0 :::2380                 :::*                    LISTEN      1396/etcd       
tcp6       0      0 :::80                   :::*                    LISTEN      2221/apache2    
tcp6       0      0 :::4369                 :::*                    LISTEN      1828/epmd       
tcp6       0      0 :::22                   :::*                    LISTEN      1619/sshd       
tcp6       0      0 ::1:6010                :::*                    LISTEN      2518/0          
root@controller:~# 
```

看到上面，主要就是 mysql , memcached, etcd 三个进程使用了 192.168.100.50，现在我们要把这些IP, 更换成 192.168.0.51

到 https://docs.openstack.org/install-guide/environment.html 找到相应的配置文件，修改这些配置至新的controller node IPv6

如mysql 就是 https://docs.openstack.org/install-guide/environment-sql-database-ubuntu.html

mysql 替换如下：

```
root@controller:~# cat /etc/mysql/mariadb.conf.d/99-openstack.cnf 
[mysqld]

feedback=ON
innodb_use_sys_malloc = 1

bind-address = 192.168.100.50

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
root@controller:~# 
root@controller:~# grep "192.168.100.50" -rl /etc/mysql/mariadb.conf.d/99-openstack.cnf | xargs sed -i "s/192.168.100.50/192.168.0.51/g"
root@controller:~# cat /etc/mysql/mariadb.conf.d/99-openstack.cnf 
[mysqld]

feedback=ON
innodb_use_sys_malloc = 1

bind-address = 192.168.0.51

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
root@controller:~# 
```
memcached 替换如下：
```
root@controller:~# grep "192.168.100.50" -rl /etc/memcached.conf | xargs sed -i "s/192.168.100.50/192.168.0.51/g"
root@controller:~# grep "192.168.100.50" -rl /etc/memcached.conf 
root@controller:~# grep "192.168.0.51" -rl /etc/memcached.conf 
/etc/memcached.conf
root@controller:~# service memcached restart
root@controller:~# service memcached status
● memcached.service - memcached daemon
   Loaded: loaded (/lib/systemd/system/memcached.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 11:33:20 CST; 3s ago
 Main PID: 5871 (memcached)
    Tasks: 6
   Memory: 480.0K
      CPU: 5ms
   CGroup: /system.slice/memcached.service
           └─5871 /usr/bin/memcached -m 64 -p 11211 -u memcache -l 192.168.0.51

Jan 10 11:33:20 controller systemd[1]: Started memcached daemon.
root@controller:~# 

etcd 替换如下：

root@controller:~# grep "192.168.100.50" -rl /etc/default/etcd | xargs sed -i "s/192.168.100.50/192.168.0.51/g"
root@controller:~# systemctl restart etcd
root@controller:~# systemctl status etcd
● etcd.service - etcd - highly-available key value store
   Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 11:35:51 CST; 5s ago
     Docs: https://github.com/coreos/etcd
           man:etcd
 Main PID: 5922 (etcd)
    Tasks: 10
   Memory: 6.7M
      CPU: 248ms
   CGroup: /system.slice/etcd.service
           └─5922 /usr/bin/etcd

Jan 10 11:35:51 controller etcd[5922]: a591474479d0011e became follower at term 20
Jan 10 11:35:51 controller etcd[5922]: newRaft a591474479d0011e [peers: [a591474479d0011e], term: 20, commit: 117159, applied: 110011, lastindex: 117159, lastterm: 20]
Jan 10 11:35:51 controller etcd[5922]: starting server... [version: 2.2.5, cluster version: 2.2]
Jan 10 11:35:51 controller systemd[1]: Started etcd - highly-available key value store.
Jan 10 11:35:51 controller etcd[5922]: a591474479d0011e is starting a new election at term 20
Jan 10 11:35:51 controller etcd[5922]: a591474479d0011e became candidate at term 21
Jan 10 11:35:51 controller etcd[5922]: a591474479d0011e received vote from a591474479d0011e at term 21
Jan 10 11:35:51 controller etcd[5922]: a591474479d0011e became leader at term 21
Jan 10 11:35:51 controller etcd[5922]: raft.node: a591474479d0011e elected leader a591474479d0011e at term 21
Jan 10 11:35:51 controller etcd[5922]: published {Name:controller ClientURLs:[http://192.168.0.51:2379]} to cluster 18c3b45aeb9b5e12
root@controller:~# 
```
做了这些更换，再检查一下：
```
root@controller:/home/ubuntu# netstat -tlnp 
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name  
tcp        0      0 192.168.100.50:3306     0.0.0.0:*               LISTEN      2064/mysqld     
tcp        0      0 192.168.0.51:2379       0.0.0.0:*               LISTEN      5922/etcd       
tcp        0      0 192.168.0.51:11211      0.0.0.0:*               LISTEN      5871/memcached   
root@controller:/home/ubuntu# 
```
发现mysql 还是没有更换成功。为什么？

再重启
```
root@controller:~# service mysql restart
root@controller:~# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name  
tcp        0      0 192.168.100.50:3306     0.0.0.0:*               LISTEN      2064/mysqld  
root@controller:~# 
```
涛声依旧
```
root@controller:~# service mysql stop
root@controller:~# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name  
tcp        0      0 192.168.100.50:3306     0.0.0.0:*               LISTEN      2064/mysqld  
root@controller:~#
```
涛声依旧

这只能够 kill -9 了。
``` 
root@controller:~# kill -9 2064
root@controller:~# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name 
tcp        0      0 192.168.0.51:3306       0.0.0.0:*               LISTEN      6713/mysqld         
root@controller:~# 
```
这里终于正常了，有我们想要的IP。
这个时候，要再次重启一下。
```
root@controller:~# service mysql status
● mysql.service - LSB: Start and stop the mysql database server daemon
   Loaded: loaded (/etc/init.d/mysql; bad; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 11:55:02 CST; 1min 34s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 6623 ExecStop=/etc/init.d/mysql stop (code=exited, status=1/FAILURE)
  Process: 6663 ExecStart=/etc/init.d/mysql start (code=exited, status=0/SUCCESS)
    Tasks: 41
   Memory: 129.2M
      CPU: 634ms
   CGroup: /system.slice/mysql.service
           ├─1890 /bin/bash /usr/bin/mysqld_safe
           ├─6713 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib/mysql/plugin --user=mysql --skip-log-error --pid-file=/var/run/mysqld/mysqld.pid --socke
           └─6714 logger -t mysqld -p daemon error

Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB: Completed initialization of buffer pool
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB: Highest supported file format is Barracuda.
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB: The log sequence numbers 28098537 and 28098537 in ibdata files do not match the log sequence number 29193113 in t
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB: Restoring possible half-written data pages from the doublewrite buffer...
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB: 128 rollback segment(s) are active.
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB: Waiting for purge to start
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] InnoDB:  Percona XtraDB (http://www.percona.com) 5.6.39-83.1 started; log sequence number 29193113
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] Server socket created on IP: '192.168.0.51'.
Jan 10 11:55:41 controller mysqld[6714]: 190110 11:55:41 [Note] /usr/sbin/mysqld: ready for connections.
Jan 10 11:55:41 controller mysqld[6714]: Version: '10.0.36-MariaDB-0ubuntu0.16.04.1'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  Ubuntu 16.04
root@controller:~# service mysql restart
root@controller:~# service mysql status
● mysql.service - LSB: Start and stop the mysql database server daemon
   Loaded: loaded (/etc/init.d/mysql; bad; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 11:56:49 CST; 1s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 6783 ExecStop=/etc/init.d/mysql stop (code=exited, status=1/FAILURE)
  Process: 6808 ExecStart=/etc/init.d/mysql start (code=exited, status=0/SUCCESS)
    Tasks: 42
   Memory: 129.3M
      CPU: 39ms
   CGroup: /system.slice/mysql.service
           ├─1890 /bin/bash /usr/bin/mysqld_safe
           ├─6713 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib/mysql/plugin --user=mysql --skip-log-error --pid-file=/var/run/mysqld/mysqld.pid --socke
           └─6714 logger -t mysqld -p daemon error

Jan 10 11:56:49 controller systemd[1]: mysql.service: Failed with result 'exit-code'.
Jan 10 11:56:49 controller systemd[1]: Starting LSB: Start and stop the mysql database server daemon...
Jan 10 11:56:49 controller mysql[6808]:  * Starting MariaDB database server mysqld
Jan 10 11:56:49 controller mysql[6808]:    ...done.
Jan 10 11:56:49 controller systemd[1]: Started LSB: Start and stop the mysql database server daemon.
root@controller:~# 
root@controller:~# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 21
Server version: 10.0.36-MariaDB-0ubuntu0.16.04.1 Ubuntu 16.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
MariaDB [(none)]> Bye
root@controller:~# 
```

现在再回到刚刚的 https://docs.openstack.org/swift/queens/install/verify.html
```
root@controller:~# swift stat
Auth version 1.0 requires ST_AUTH, ST_USER, and ST_KEY environment variables
to be set or overridden with -A, -U, or -K.

Auth version 2.0 requires OS_AUTH_URL, OS_USERNAME, OS_PASSWORD, and
OS_TENANT_NAME OS_TENANT_ID to be set or overridden with --os-auth-url,
--os-username, --os-password, --os-tenant-name or os-tenant-id. Note:
adding "-V 2" is necessary for this.
root@controller:~# 
```

这里又是另一个问题了哈！~ 
```
root@controller:~# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 13:17:11 CST; 13s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 8761 ExecStop=/etc/init.d/swift-proxy stop (code=exited, status=0/SUCCESS)
  Process: 8773 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 88.2M
      CPU: 1.200s
   CGroup: /system.slice/swift-proxy.service
           ├─8784 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─8793 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─8794 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─8795 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─8796 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Jan 10 13:17:21 controller proxy-server[8796]: ERROR Insufficient Storage 192.168.0.180:6202/sdc (txn: tx052351248e9443ff9d8f8-005c36d561)
Jan 10 13:17:21 controller proxy-server[8796]: ERROR Insufficient Storage 192.168.0.135:6202/sdc (txn: tx052351248e9443ff9d8f8-005c36d561)
Jan 10 13:17:21 controller proxy-server[8796]: Account HEAD returning 503 for [507, 507, 507] (txn: tx052351248e9443ff9d8f8-005c36d561)
Jan 10 13:17:21 controller proxy-server[8796]: - - 10/Jan/2019/05/17/21 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 503 - Swift - - - - tx052351248e9443ff9d8f8-
Jan 10 13:17:23 controller proxy-server[8796]: Account HEAD returning 503 for [] (txn: tx052351248e9443ff9d8f8-005c36d561) (client_ip: 192.168.0.51)
Jan 10 13:17:23 controller proxy-server[8796]: 192.168.0.51 192.168.0.51 10/Jan/2019/05/17/23 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 503 - python-swiftclie
Jan 10 13:17:24 controller proxy-server[8796]: Account HEAD returning 503 for [] (txn: tx409727b312c54262bb43a-005c36d564)
Jan 10 13:17:24 controller proxy-server[8796]: - - 10/Jan/2019/05/17/24 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 503 - Swift - - - - tx409727b312c54262bb43a-
Jan 10 13:17:24 controller proxy-server[8796]: Account HEAD returning 503 for [] (txn: tx409727b312c54262bb43a-005c36d564) (client_ip: 192.168.0.51)
Jan 10 13:17:24 controller proxy-server[8796]: 192.168.0.51 192.168.0.51 10/Jan/2019/05/17/24 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 503 - python-swiftclie
root@controller:~# 
```
看这个问题，应该是与 "ERROR Insufficient Storage 192.168.0.180:6202/sdc" 相关了。一看就知道了，我们这里是配置错了，应该是 sdb 而不是 sdc 。所以，应该是我们在配置 ring.gz 的时候弄错了！

重复 https://docs.openstack.org/swift/queens/install/initial-rings.html 过程，重新 rebalance 吧。

再来
```
root@controller:~# swift stat 
Auth version 1.0 requires ST_AUTH, ST_USER, and ST_KEY environment variables
to be set or overridden with -A, -U, or -K.

Auth version 2.0 requires OS_AUTH_URL, OS_USERNAME, OS_PASSWORD, and
OS_TENANT_NAME OS_TENANT_ID to be set or overridden with --os-auth-url,
--os-username, --os-password, --os-tenant-name or os-tenant-id. Note:
adding "-V 2" is necessary for this.
root@controller:~# 
```
这个问题是在 controller node 中的 /etc/swift/proxy-server.conf 中的 auth_url和memcached_servers配置的问题，配置成下面这样就可以了。
```
root@controller:~# grep controller -rn /etc/swift/proxy-server.conf 
365:www_authenticate_uri = http://controller:5000
366:auth_url = http://controller:5000
367:memcached_servers = controller:11211
472:memcache_servers = controller:11211
root@controller:~# 
```
重新启动 swift-proxy 。

这时候，果然成功了。
```
root@controller:~# . demo-openrc 
root@controller:~# swift stat
               Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
            Containers: 0
               Objects: 0
                 Bytes: 0
       X-Put-Timestamp: 1547100678.74534
           X-Timestamp: 1547100678.74534
            X-Trans-Id: txc0d3298c805c45a9afb35-005c36e203
          Content-Type: text/plain; charset=utf-8
X-Openstack-Request-Id: txc0d3298c805c45a9afb35-005c36e203
root@controller:~# 
```
但是在创建 container 的时候出错了！
```
root@controller:~# openstack container create container1
Internal Server Error (HTTP 500) (Request-ID: txb93d09f0ee1a4e9fbe9c7-005c36e410)
root@controller:~# service swift-proxy status
● swift-proxy.service - LSB: Swift proxy server
   Loaded: loaded (/etc/init.d/swift-proxy; bad; vendor preset: enabled)
   Active: active (running) since Thu 2019-01-10 14:10:23 CST; 9min ago
     Docs: man:systemd-sysv-generator(8)
  Process: 9674 ExecStop=/etc/init.d/swift-proxy stop (code=exited, status=0/SUCCESS)
  Process: 9686 ExecStart=/etc/init.d/swift-proxy start (code=exited, status=0/SUCCESS)
    Tasks: 5
   Memory: 98.8M
      CPU: 9.731s
   CGroup: /system.slice/swift-proxy.service
           ├─9697 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─9706 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─9707 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           ├─9708 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
           └─9709 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf

Jan 10 14:12:58 controller proxy-server[9706]: Container GET returning 503 for (503, 503, 503) (txn: tx8f9df230a2f544808d3a5-005c36e26a) (client_ip: 192.168.0.51)
Jan 10 14:12:58 controller proxy-server[9706]: Could not autocreate account '/AUTH_7d6eaa90d74a4f239963933c3a744df3' (txn: tx8f9df230a2f544808d3a5-005c36e26a) (client_ip: 192.168.0.51)
Jan 10 14:12:58 controller proxy-server[9706]: 192.168.0.51 192.168.0.51 10/Jan/2019/06/12/58 PUT /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/container1 HTTP/1.0 500 - python-swiftclient-3.5
Jan 10 14:20:01 controller proxy-server[9708]: - - 10/Jan/2019/06/20/01 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 200 - Swift - - - - txb93d09f0ee1a4e9fbe9c7-
Jan 10 14:20:01 controller proxy-server[9708]: ERROR 500 Trying to PUT /AUTH_7d6eaa90d74a4f239963933c3a744df3 From Container Server 192.168.0.198:6202/sdb (txn: txb93d09f0ee1a4e9fbe9c7-005
Jan 10 14:20:01 controller proxy-server[9708]: ERROR 500 Trying to PUT /AUTH_7d6eaa90d74a4f239963933c3a744df3 From Container Server 192.168.0.135:6202/sdb (txn: txb93d09f0ee1a4e9fbe9c7-005
Jan 10 14:20:01 controller proxy-server[9708]: ERROR 500 Trying to PUT /AUTH_7d6eaa90d74a4f239963933c3a744df3 From Container Server 192.168.0.180:6202/sdb (txn: txb93d09f0ee1a4e9fbe9c7-005
Jan 10 14:20:01 controller proxy-server[9708]: Container GET returning 503 for (503, 503, 503) (txn: txb93d09f0ee1a4e9fbe9c7-005c36e410) (client_ip: 192.168.0.51)
Jan 10 14:20:01 controller proxy-server[9708]: Could not autocreate account '/AUTH_7d6eaa90d74a4f239963933c3a744df3' (txn: txb93d09f0ee1a4e9fbe9c7-005c36e410) (client_ip: 192.168.0.51)
Jan 10 14:20:01 controller proxy-server[9708]: 192.168.0.51 192.168.0.51 10/Jan/2019/06/20/01 PUT /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3/container1 HTTP/1.0 500 - osc-lib/1.9.0%20keysto

root@controller:~# 
```
这个就，搞大发了，又出错了。要不要这样对我！~
```
ERROR 500 Trying to PUT /AUTH_7d6eaa90d74a4f239963933c3a744df3 From Container Server 192.168.0.198:6202/sdb
```
这里说的是 "Container Server 192.168.0.198:6202" , 但是正确的对应是下面这样的：
```
object.builder 6200 
container.builder 6201 
account.builder 6202
```
为什么？

然后，没有思路了。

选择重新启动所有的node。

但是，其中2个storage node 挂了，要重新烧一下盘！~悲剧！

按照肖工给的流程，重新烧好了！

然后，重新配置 storage node ,然后上面这个问题就消失了，哈哈。

那就可以了！




