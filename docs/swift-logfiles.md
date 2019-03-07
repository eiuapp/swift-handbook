

## 当 swift 出问题时，我们应该看哪些日志文件

### apache2 相关： 

- /var/log/apache2/access.log
- /var/log/apache2/error.log

### keystone 相关：

#### 进入 apache2/keystone 

- /var/log/apache2/keystone_access.log 
- /var/log/apache2/keystone.log

#### keystone 自己日志
- /var/log/keystone/keystone-wsgi-public.log

### swift 相关: 

- /var/log/syslog 

### rsyncd 相关：

- /var/log/rsyncd.log


### mysql 相关：

如果没有配置log dir,则 log 在系统log（`/var/log/syslog`）中。
