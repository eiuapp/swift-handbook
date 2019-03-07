
How to enable Reseller Admin for OpenStack Swift Object Storage and KeyStone V3?

本文整理自 https://www.ibm.com/developerworks/community/forums/html/topic?id=813eb4fd-c0c6-43a5-9317-a35e4081ef72

How to enable Reseller Admin for OpenStack Swift Object Storage and KeyStone V3?

Jul 2, 2015 | Tags: account, admin, configuring, enable, enabling, how, keystone, management, object, openstack, reseller, reseller_admin, reselleradmin, scale, spectrum, swift, to, v3

How to enable Reseller Admin for OpenStack Swift Object Storage and KeyStone V3?

The purpose of the article is to leverage the Reseller Admin concept in Swift to query account statistics from a single admin level user.  As per OpenStack Swift documentation, Users with the Keystone role defined in reseller_admin_role (ResellerAdmin by default) can operate on any account. The auth system sets the request environ reseller_request to True if a request is coming from a user with this role.

This article explains how this can be configured. In this sample exercise, our goal is to enable Reseller Admin access to 'admin' user belonging to 'admin' project.  You can also create other users and enable ResellerAdmin access.

 

Assumptions:

KeyStone V3 is setup and Swift is configured. 
Python openstack client command line interface is available.
By default, 'admin' user and 'admin' project is created and assigned to 'default' Domain.  We will use this in this exercise, however you can configure different user, project and domain to accomplish ResellerAdmin setup.
Openrc file is available to work with openstack CLI client.
 

Initially, you will see something like this

```shell
 [root@host ~]# cat openrc
# Mon Jun 29 03:34:36 EDT 2015
export OS_AUTH_URL="http://127.0.0.1:35357/v3"
export OS_IDENTITY_API_VERSION=3
export OS_AUTH_VERSION=3
export OS_USERNAME="admin"
export OS_PASSWORD="password"
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_PROJECT_DOMAIN_NAME=Default
[root@host ~]# openstack domain list
+---------+---------+---------+----------------------------------------------------------------------+
| ID      | Name    | Enabled | Description                                                          
+---------+---------+---------+----------------------------------------------------------------------+
| default | Default | True    | Owns users and tenants (i.e. projects) available on Identity API v2. |
+---------+---------+---------+----------------------------------------------------------------------+
```
 
Now, create a new Domain

```shell
[root@host ~]# openstack domain create Domain1
+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | 193bed5ba5a5481c8676bb5e06cf6125 |
| name    | Domain1                          |
+---------+----------------------------------+
[root@host ~]# openstack domain list
+----------------------------------+---------+---------+----------------------------------------------------------------------+
| ID                               | Name    | Enabled | Description                                                          
+----------------------------------+---------+---------+----------------------------------------------------------------------+
| default                          | Default | True    | Owns users and tenants (i.e. projects) available on Identity API v2. |
| 193bed5ba5a5481c8676bb5e06cf6125 | Domain1 | True    |                                                                      |
+----------------------------------+---------+---------+----------------------------------------------------------------------+
``` 
 
Next, create a new project, lets call it 'Project1'

```shell
[root@host ~]# openstack project list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 534bca56aca443e889036ab52cfe40ec | admin   |
| 4c03ae000dd54860a4f9645d75272f5e | service |
+----------------------------------+---------+
[root@host ~]# openstack project create Project1 --domain Domain1
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | 193bed5ba5a5481c8676bb5e06cf6125 |
| enabled     | True                             |
| id          | e597d73f169444a1bb55a1e3a4adb29d |
| name        | Project1                         |
+-------------+----------------------------------+
[root@host ~]# openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 534bca56aca443e889036ab52cfe40ec | admin    |
| 4c03ae000dd54860a4f9645d75272f5e | service  |
| e597d73f169444a1bb55a1e3a4adb29d | Project1 |
+----------------------------------+----------+
```

Next, create a new user 'User1' and assign him to 'Project1' as part of 'Domain1'

```shell
[root@host ~]#  openstack user create --domain Domain1 --project Project1 --password User1 User1
+--------------------+----------------------------------+
| Field              | Value                            |
+--------------------+----------------------------------+
| default_project_id | e597d73f169444a1bb55a1e3a4adb29d |
| domain_id          | 193bed5ba5a5481c8676bb5e06cf6125 |
| enabled            | True                             |
| id                 | d82e44a986234ad9bbac48d58cd2d18c |
| name               | User1                            |
+--------------------+----------------------------------+
``` 
 

In KeyStone V3, by default, you will see  'admin' role is being configured and available

```shell
[root@host ~]# openstack role list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 4fbfdce0a56b47dd8d29907f534d4b52 | admin |
+----------------------------------+-------+
``` 

Lets add the newly created 'User1' to 'Project1' as part of 'Domain1'
 
```shell
[root@host ~]# openstack role add --project Project1 --user User1 admin
[root@host ~]# echo $?
0
[root@host ~]# openstack role add --domain Domain1 --user User1 admin
[root@host ~]# echo $?
0
```

Copy openrc and modify it to create  a testrc (sample below)

```shell
[root@host ~]# cat testrc
# Mon Jun 29 03:34:36 EDT 2015
export OS_AUTH_URL="http://127.0.0.1:35357/v3"
export OS_IDENTITY_API_VERSION=3
export OS_AUTH_VERSION=3
export OS_USERNAME="User1"
export OS_PASSWORD="User1"
export OS_USER_DOMAIN_NAME=Domain1
export OS_PROJECT_NAME=Project1
export OS_PROJECT_DOMAIN_NAME=Domain1
```

Source the 'testrc', we will try and create a swift container and upload a sample object and get account level statistics.
 
```shell
[root@host ~]# source testrc
``` 

Intially, 'swift list' shows nothing.

```shell
[root@host ~]# swift list
[root@host ~]#
``` 
 
Now, create a container

```shell
[root@host ~]# swift post testcontainer1
``` 

We can upload an object to the 'testcontainer1'

```shell
[root@host ~]# swift upload testcontainer1 file1
file1
[root@host ~]# swift list
testcontainer1
``` 

Now, ensure swift stat shows

```shell
[root@host ~]# swift stat
                        Account: AUTH_e597d73f169444a1bb55a1e3a4adb29d
                     Containers: 1
                        Objects: 1
                          Bytes: 166978
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 166978
    X-Account-Project-Domain-Id: 193bed5ba5a5481c8676bb5e06cf6125
                    X-Timestamp: 1435818271.19603
                     X-Trans-Id: txa197a1317218450c8a734-005594e2ec
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
[root@host ~]# swift list testcontainer1
file1
``` 

Now, source 'openrc' to 'admin' user of 'admin' project with 'Default' domain
 
```shell
[root@host ~]# source openrc
``` 

As 'admin' user, If you try to list the container created by 'User1' of 'Project1' and 'Domain1', you will not be able to do this.  This requires ResellerAdmin level access, which is what we are going to configure now.

```shell
[root@host ~]# swift list testcontainer1
Container 'testcontainer1' not found
``` 

Swift proxy-server.conf needs to be edited to add 'reseller_admin_role' and 'reseller_prefix'.

As per KeyStone V3 documentation, the name 'ResellerAdmin' is not hardcoded and can be user defined.  
'reseller_prefix' can be set to a user defined prefix.  We will use 'AUTH_', but you can use something else as well.


By default, on /etc/swift/proxy-server.conf, you will see something like this

```
[filter:keystone]
use = egg:swift#keystoneauth
operator_roles = admin, SwiftOperator
is_admin = true
cache = swift.cache
``` 
 

Edit the  proxy-server.conf (filter:keystone section) and add the lines highlighted in bold.
``` 
[filter:keystone]
use = egg:swift#keystoneauth
operator_roles = admin, SwiftOperator
reseller_admin_role = ResellerAdmin
reseller_prefix = AUTH_
is_admin = true
cache = swift.cache
``` 
 

Now, create ResellerAdmin role.
```shell
[root@host ~]# openstack role create ResellerAdmin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 9658c423949a460bbb6ed04b19fae672 |
| name  | ResellerAdmin                    |
+-------+----------------------------------+
[root@host ~]# openstack role list
+----------------------------------+---------------+
| ID                               | Name          |
+----------------------------------+---------------+
| 4fbfdce0a56b47dd8d29907f534d4b52 | admin         |
| 9658c423949a460bbb6ed04b19fae672 | ResellerAdmin |
+----------------------------------+---------------+
``` 
 
Add 'ResellerAdmin' role to 'admin' user of 'admin' project with 'Default' domain.

```shell
[root@host ~]# openstack role add --domain Default --user admin ResellerAdmin
[root@host ~]# openstack role add  --project admin --user admin ResellerAdmin
``` 
 
The following command confirms, that 'ResellerAdmin' role is assigned to 'admin' user.

```shell
[root@host ~]# openstack role assignment list --role ResellerAdmin
+----------------------------------+----------------------------------+-------+----------------------------------+---------+
| Role                             | User                             | Group | Project                          | Domain  |
+----------------------------------+----------------------------------+-------+----------------------------------+---------+
| 9658c423949a460bbb6ed04b19fae672 | 56dbaf1aa80247c7a4834c0fea604926 |       |                                 | default |
| 9658c423949a460bbb6ed04b19fae672 | 56dbaf1aa80247c7a4834c0fea604926 |       | 534bca56aca443e889036ab52cfe40ec |         |
+----------------------------------+----------------------------------+-------+----------------------------------+---------+

[root@host ~]# openstack user show 56dbaf1aa80247c7a4834c0fea604926
+--------------------+----------------------------------+
| Field              | Value                            |
+--------------------+----------------------------------+
| default_project_id | 534bca56aca443e889036ab52cfe40ec |
| domain_id          | default                          |
| enabled            | True                             |
| id                 | 56dbaf1aa80247c7a4834c0fea604926 |
| name               | admin                            |
+--------------------+----------------------------------+

[root@host ~]# openstack project show 534bca56aca443e889036ab52cfe40ec
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 534bca56aca443e889036ab52cfe40ec |
| name        | admin                            |
+-------------+----------------------------------+
``` 
 
As 'admin' user, invoke keystone to get a new token.

```shell
[root@host ~]# export TOKEN=`openstack --os-username admin --os-project-name admin  token issue | grep id | head -n 1 | awk '{print $4}' `
[root@host ~]# echo $TOKEN
a5957db94c3e4fb6aa2f118cbdf63896
``` 

You have provided 'ResellerAdmin' access to user 'admin'.  Find, the project Id that you want to access using 'ResellerAdmin' access.

```shell
[root@host ~]# openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 534bca56aca443e889036ab52cfe40ec | admin    |
| 4c03ae000dd54860a4f9645d75272f5e | service  |
| e597d73f169444a1bb55a1e3a4adb29d | Project1 |
+----------------------------------+----------+
``` 
 

Now, as 'admin' user from 'admin' project of 'Default' domain, get the account stat for Project1.

Remember, we added 'AUTH_' as the reseller prefix in proxy-server.conf.  So append the project id to 'AUTH_' and issue the following openstack command.   The following swift command line shows how to invoke the account stats using the reseller prefix.

 
Note, the swift endpoint URI can be obtained via 'openstack endpoint list'.

```shell
[root@host ~]# swift --os-auth-token $TOKEN --os-storage-url http://localhost:8080/v1/AUTH_e597d73f169444a1bb55a1e3a4adb29d stat
                        Account: AUTH_e597d73f169444a1bb55a1e3a4adb29d
                     Containers: 1
                        Objects: 1
                          Bytes: 166978
Containers in policy "policy-0": 1
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 166978
    X-Account-Project-Domain-Id: 193bed5ba5a5481c8676bb5e06cf6125
                    X-Timestamp: 1435818271.19603
                     X-Trans-Id: tx2d07e74417ca4e0abea68-005594dda2
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
[root@host ~]# swift --os-auth-token $TOKEN --os-storage-url http://localhost:8080/v1/AUTH_e597d73f169444a1bb55a1e3a4adb29d list
testcontainer1
[root@host ~]# swift --os-auth-token $TOKEN --os-storage-url http://localhost:8080/v1/AUTH_e597d73f169444a1bb55a1e3a4adb29d list testcontainer1
file1
``` 
 
Now, you are able to query any account using a single user using 'ResellerAdmin' level access, without requiring password of those accounts.  

Happy OpenStack Swift!!!

Note: The information in this blog is provided "AS IS".  The opinions expressed on this blog represent my own and not those of my employer.

Log in to reply.
Updated on Jul 2, 2015 at 11:21 PM by manbazha