
本文记录 swift 配置多proxy节点过程中出现的问题

## 关闭 proxy node 的swift-proxy 后，proxybak node 也失效

#### 现场

出现下面的2个错误：

- Account GET failed: http://swiftproxy:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json 500 Internal Error   An error occurred
- HTTPConnectionPool(host='swiftproxy', port=8080): Max retries exceeded with url: /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 (Caused by NewConnectionError('<requests.packages.urllib3.connection.HTTPConnection object at 0x7f6345ef4290>: Failed to establish a new connection: [Errno 111] Connection refused',))

第一个问题是与认证相关；第二个问题是服务不能正常访问。

#### proxy node 
```shell
root@controller:~# service swift-proxy stop
```

#### proxybak node 

```shell
root@ubuntu:/etc/swift# swift list
HTTPConnectionPool(host='controller', port=8080): Max retries exceeded with url: /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json (Caused by NewConnectionError('<requests.packages.urllib3.connection.HTTPConnection object at 0x7ffb6b334090>: Failed to establish a new connection: [Errno 111] Connection refused',))
root@ubuntu:/etc/swift#
```

为什么会这样？按道理，这个时候，应该正常，可以访问的。

这里显示是连接到 controller 的 8080 ，这个地方是哪里来的呢？这个当然就是openstack endpoint注册的 swift 的 三个endpoint 之一了。

回到一台 能访问 openstack endpoint 的机器，查一下。

#### （本节，不要操作）更新 swift服务的endpoint

然后，重新添加一个新的 proxybak 地址的 swift endpoint。（注意：实际上是覆盖，而不是增加）

```shell
openstack endpoint create --region RegionOne \
  object-store public http://swiftproxy:8080/v1/AUTH_%\(project_id\)s 

openstack endpoint create --region RegionOne \
  object-store internal http://swiftproxy:8080/v1/AUTH_%\(project_id\)s 

openstack endpoint create --region RegionOne \
  object-store admin http://swiftproxy:8080/v1
```

现在就有2个swift服务的URL，效果如下：

```shell
root@controller:~# . admin-openrc 
root@controller:~# openstack endpoint list | grep swift
| 057cd51c8ace4daa8834d25ae15998f4 | RegionOne | swift        | object-store | True    | admin     | http://controller:8080/v1                     |
| 51952812b675436899ba118585b2dae2 | RegionOne | swift        | object-store | True    | public    | http://swiftproxy:8080/v1/AUTH_%(project_id)s |
| 87de9f6afbd74792bc80c7d092eb7da6 | RegionOne | swift        | object-store | True    | internal  | http://controller:8080/v1/AUTH_%(project_id)s |
| beccdad7270643cabf8a271258c102c0 | RegionOne | swift        | object-store | True    | admin     | http://swiftproxy:8080/v1                     |
| d3f5808482cb42f386c3741b8e1c1d82 | RegionOne | swift        | object-store | True    | public    | http://controller:8080/v1/AUTH_%(project_id)s |
| e577371ea7e349ffa2a86f7aef2ef35e | RegionOne | swift        | object-store | True    | internal  | http://swiftproxy:8080/v1/AUTH_%(project_id)s |
root@controller:~# 
``` 

#### 再次访问

```shell
root@controller:~# swift list
Account GET failed: http://swiftproxy:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3?format=json 500 Internal Error   An error occurred
Failed Transaction ID: txeaebd48bfe1346dc90161-005c481bf9
root@controller:~# cd /etc/swift/
root@ubuntu:/etc/swift# ping swiftproxy
PING swiftproxy (192.168.0.141) 56(84) bytes of data.
64 bytes from swiftproxy (192.168.0.141): icmp_seq=1 ttl=64 time=0.025 ms
^C
--- swiftproxy ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.025/0.025/0.025/0.000 ms
root@ubuntu:/etc/swift# 
```

#### 查看 swift-proxy 日志

swift-proxy 日志是系统日志 /var/log/syslog

这里看到了报错如下：

```text
Jan 23 16:00:09 ubuntu proxy-server: Deferring reject downstream
Jan 23 16:00:10 ubuntu proxy-server: - - 23/Jan/2019/08/00/10 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.0 204 - Swift - - - - txf2bd6100da9d4e45bb421-005c481f09 - 0.0574 RL - 1548230409.998038054 1548230410.055468082 -
Jan 23 16:00:10 ubuntu proxy-server: 192.168.0.141 192.168.0.141 23/Jan/2019/08/00/10 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.0 500 - python-swiftclient-3.0.0 gAAAAABcSB8JuXYf... - - - txf2bd6100da9d4e45bb421-005c481f09 - 0.0734 - - 1548230409.982681036 1548230410.056046963 -
Jan 23 16:00:10 ubuntu proxy-server: Error: An error occurred: #012Traceback (most recent call last):#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/catch_errors.py", line 41, in handle_request#012    resp = self._app_call(env)#012  File "/usr/lib/python2.7/dist-packages/swift/common/wsgi.py", line 1038, in _app_call#012    resp = self.app(env, self._start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/gatekeeper.py", line 99, in __call__#012    return self.app(env, gatekeeper_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/healthcheck.py", line 57, in __call__#012    return self.app(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/proxy_logging.py", line 346, in __call__#012    six.reraise(exc_type, exc_value, exc_traceback)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/proxy_logging.py", line 338, in __call__#012    iterable = self.app(env, my_start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/memcache.py", line 109, in __call__#012    return self.app(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/swob.py", line 1386, in _wsgify_self#012    return func(self, Request(env))(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/swob.py", line 1386, in _wsgify_self#012    return func(self, Request(env))(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/ratelimit.py", line 301, in __call__#012    return self.app(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/webob/dec.py", line 130, in __call__#012    resp = self.call_func(req, *args, **self.kwargs)#012  File "/usr/lib/python2.7/dist-packages/webob/dec.py", line 195, in call_func#012    return self.func(req, *args, **kwargs)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 464, in __call__#012    response = self.process_request(req)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 732, in process_request#012    resp = super(AuthProtocol, self).process_request(request)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 492, in process_request#012    data, user_auth_ref = self._do_fetch_token(request.user_token)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 531, in _do_fetch_token#012    data = self.fetch_token(token)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 835, in fetch_token#012    cached = self._cache_get_hashes(token_hashes)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 818, in _cache_get_hashes#012    cached = self._token_cache.get(token)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/_cache.py", line 222, in get#012    with self._cache_pool.reserve() as cache:#012  File "/usr/lib/python2.7/contextlib.py", line 17, in __enter__#012    return self.gen.next()#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/_cache.py", line 77, in reserve#012    c = memorycache.get_client(self._memcached_servers)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/openstack/common/memorycache.py", line 44, in get_client#012    import memcache#012ImportError: No module named memcache (txn: txf2bd6100da9d4e45bb421-005c481f09)
Jan 23 16:00:11 ubuntu proxy-server: 192.168.0.141 192.168.0.141 23/Jan/2019/08/00/11 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.0 500 - python-swiftclient-3.0.0 gAAAAABcSB8JuXYf... - - - tx2336d108a1b94b8ba1f39-005c481f0b - 0.0214 - - 1548230411.059453964 1548230411.080816031 -
Jan 23 16:00:11 ubuntu proxy-server: Error: An error occurred: #012Traceback (most recent call last):#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/catch_errors.py", line 41, in handle_request#012    resp = self._app_call(env)#012  File "/usr/lib/python2.7/dist-packages/swift/common/wsgi.py", line 1038, in _app_call#012    resp = self.app(env, self._start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/gatekeeper.py", line 99, in __call__#012    return self.app(env, gatekeeper_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/healthcheck.py", line 57, in __call__#012    return self.app(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/proxy_logging.py", line 346, in __call__#012    six.reraise(exc_type, exc_value, exc_traceback)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/proxy_logging.py", line 338, in __call__#012    iterable = self.app(env, my_start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/memcache.py", line 109, in __call__#012    return self.app(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/swob.py", line 1386, in _wsgify_self#012    return func(self, Request(env))(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/swob.py", line 1386, in _wsgify_self#012    return func(self, Request(env))(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/swift/common/middleware/ratelimit.py", line 301, in __call__#012    return self.app(env, start_response)#012  File "/usr/lib/python2.7/dist-packages/webob/dec.py", line 130, in __call__#012    resp = self.call_func(req, *args, **self.kwargs)#012  File "/usr/lib/python2.7/dist-packages/webob/dec.py", line 195, in call_func#012    return self.func(req, *args, **kwargs)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 464, in __call__#012    response = self.process_request(req)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 732, in process_request#012    resp = super(AuthProtocol, self).process_request(request)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 492, in process_request#012    data, user_auth_ref = self._do_fetch_token(request.user_token)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 531, in _do_fetch_token#012    data = self.fetch_token(token)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 835, in fetch_token#012    cached = self._cache_get_hashes(token_hashes)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/__init__.py", line 818, in _cache_get_hashes#012    cached = self._token_cache.get(token)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/_cache.py", line 222, in get#012    with self._cache_pool.reserve() as cache:#012  File "/usr/lib/python2.7/contextlib.py", line 17, in __enter__#012    return self.gen.next()#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/auth_token/_cache.py", line 77, in reserve#012    c = memorycache.get_client(self._memcached_servers)#012  File "/usr/lib/python2.7/dist-packages/keystonemiddleware/openstack/common/memorycache.py", line 44, in get_client#012    import memcache#012ImportError: No module named memcache (txn: tx2336d108a1b94b8ba1f39-005c481f0b)
```

在这里可以看到 `import memcache#012ImportError: No module named memcache`。所以应该是python执行`import memcache` 没有成功。

#### python 安装 python-memcached

```shell
root@ubuntu:~# pip install python-memcached
```

#### 再来一次

```shell
root@ubuntu:/etc/swift# . demo-openrc 
root@ubuntu:/etc/swift# swift stat -v
                     StorageURL: http://swiftproxy:8080/v1/AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Auth Token: gAAAAABcSC01KwFvQHMmrotw_7VKFVeUhXtqmrUCD8x0HEqfHQCXQ4MX4KI-KD6Qq4b5K8J-XdoflfOJxWgFoVKWftnC5AHokX_Eyu0fIzI3oarICGXh5kSsM49r2RXInw5TRljZnBT05iRbRsPGKoCvvdxcxsCW8nzzOh_RZWs4PbPBGZSeU6g
                        Account: AUTH_7d6eaa90d74a4f239963933c3a744df3
                     Containers: 2
                        Objects: 30
                          Bytes: 3718416484
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 30
     Bytes in policy "policy-0": 3718416484
    X-Account-Project-Domain-Id: default
                    X-Timestamp: 1547188179.49086
                     X-Trans-Id: txa5bbe4ae9a1243ab84539-005c482d35
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
root@ubuntu:/etc/swift# 
```

swift-proxy 日志也正常了。

```text
pipJan 23 16:17:01 ubuntu CRON[15089]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
Jan 23 16:31:55 ubuntu proxy-server: Deferring reject downstream
Jan 23 16:31:55 ubuntu proxy-server: ERROR with Account server 192.168.0.134:6202/sdb re: Trying to HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3: ConnectionTimeout (0.5s) (txn: tx4b41c78a86b246c59725d-005c48267b)
Jan 23 16:31:56 ubuntu proxy-server: - - 23/Jan/2019/08/31/56 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.0 204 - Swift - - - - tx4b41c78a86b246c59725d-005c48267b - 1.0088 RL - 1548232315.264564991 1548232316.273391962 -
Jan 23 16:31:57 ubuntu proxy-server: 192.168.0.141 192.168.0.141 23/Jan/2019/08/31/57 HEAD /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3 HTTP/1.0 204 - python-swiftclient-3.0.0 gAAAAABcSCZ7G8AM... - - - tx4b41c78a86b246c59725d-005c48267b - 2.3865 - - 1548232315.181828022 1548232317.568300962 -
Jan 23 16:32:28 ubuntu proxy-server: 192.168.0.141 192.168.0.141 23/Jan/2019/08/32/28 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson HTTP/1.0 200 - python-swiftclient-3.0.0 gAAAAABcSCabDkEo... - 114 - tx85f8ee55b83a4dfeab891-005c48269b - 0.8730 - - 1548232347.452719927 1548232348.325676918 -
Jan 23 16:32:28 ubuntu proxy-server: ERROR with Account server 192.168.0.134:6202/sdb re: Trying to GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3: ConnectionTimeout (0.5s) (txn: txe1bf0548bc31486aafd60-005c48269c) (client_ip: 192.168.0.141)
Jan 23 16:32:28 ubuntu proxy-server: 192.168.0.141 192.168.0.141 23/Jan/2019/08/32/28 GET /v1/AUTH_7d6eaa90d74a4f239963933c3a744df3%3Fformat%3Djson%26marker%3Dcontainer2 HTTP/1.0 200 - python-swiftclient-3.0.0 gAAAAABcSCabDkEo... - 2 - txe1bf0548bc31486aafd60-005c48269c - 0.6325 - - 1548232348.327672958 1548232348.960134983 -
```

问题解决。