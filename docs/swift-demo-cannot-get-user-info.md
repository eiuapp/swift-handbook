


## env 

当demo用户登录 horizone , 访问 http://controller/horizon/identity/users/ 时 提示 "Error: 无法获取用户信息" 

## step

因为我们知道，demo是 属于:

- project: demo
- user: demo
- role: user

那么，我们只要把 demo(project)中的 demo(user)赋予 admin(role)，就可以了。

```
root@ubuntu:/home/administrator# openstack project list
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 10636c6cebef4643b336d8edee168fc7 | demo    |
| 544bf3a68de64987ae3bd92f640facc4 | admin   |
| bac2c98b90af448da37507f939f6fb33 | test1   |
| c944137b50c34ecd9f03f8911e817209 | service |
+----------------------------------+---------+
root@ubuntu:/home/administrator# openstack role list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| a9c6feb32e8d4d099e0140111c9fc137 | admin |
| dfbf44be3cfe4f4790944bfc4300a3ba | user  |
+----------------------------------+-------+
root@ubuntu:/home/administrator# openstack role show user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | dfbf44be3cfe4f4790944bfc4300a3ba |
| name      | user                             |
+-----------+----------------------------------+
root@ubuntu:/home/administrator# openstack role add --project demo --user demo admin
root@ubuntu:/home/administrator# 
```

## ref
- https://docs.openstack.org/keystone/rocky/install/keystone-openrc-ubuntu.html