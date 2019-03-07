+++
title = "openstack swift overview ring"
date = 2019-01-20T00:00:00-08:00
lastmod = 2019-01-21T18:09:25-08:00
tags = ["swift"]
categories = ["swift"]
draft = false
weight = 3002
+++

## list of devices {#list-of-devices}

<https://docs.openstack.org/swift/queens/overview%5Fring.html#list-of-devices>

这里的 list of devices 可以通过下面方式，查看得到。

```shell
root@controller:~/tmp2# cp /etc/swift/account.ring.gz  .
root@controller:~/tmp2# gzip -d ./account.ring.gz
gzip: ./account.ring already exists; do you wish to overwrite (y or n)? y
root@controller:~/tmp2# tail account.ring                                                                                                                                                                           R1NG"byteorder": "little", "devs": [{"device": "sdb", "id": 0, "ip": "192.168.0.134", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.134", "replication_port": 6202, "weight": 100.0, "zone": 1}, null, {"device": "sdb", "id": 2, "ip": "192.168.0.127", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.127", "replication_port": 6202, "weight": 100.0, "zone": 1}, {"device": "sdb", "id": 3, "ip": "192.168.0.198", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.198", "replication_port": 6202, "weight": 100.0, "zone": 1}, {"device": "sdb", "id": 4, "ip": "192.168.0.180", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.180", "replication_port": 6202, "weight": 100.0, "zone": 1}, {"device": "sdb", "id": 5, "ip": "192.168.0.135", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.135", "replication_port": 6202, "weight": 100.0, "zone": 1}], "part_shift": 22, "replica_count": 3}root@controller:~/tmp2#
R1NG{"byteorder": "little", "devs": [{"device": "sdb", "id": 0, "ip": "192.168.0.134", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.134", "replication_port": 6202, "weight": 100.0, "zone": 1}, null, {"device": "sdb", "id": 2, "ip": "192.168.0.127", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.127", "replication_port": 6202, "weight": 100.0, "zone": 1}, {"device": "sdb", "id": 3, "ip": "192.168.0.198", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.198", "replication_port": 6202, "weight": 100.0, "zone": 1}, {"device": "sdb", "id": 4, "ip": "192.168.0.180", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.180", "replication_port": 6202, "weight": 100.0, "zone": 1}, {"device": "sdb", "id": 5, "ip": "192.168.0.135", "meta": "", "port": 6202, "region": 1, "replication_ip": "192.168.0.135", "replication_port": 6202, "weight": 100.0, "zone": 1}], "part_shift": 22, "replica_count": 3}root@controller:~/tmp2#
  root@controller:~/tmp2#
```

格式化一下json，如下

```json
{
    "byteorder": "little",
    "devs": [{
        "device": "sdb",
        "id": 0,
        "ip": "192.168.0.134",
        "meta": "",
        "port": 6202,
        "region": 1,
        "replication_ip": "192.168.0.134",
        "replication_port": 6202,
        "weight": 100.0,
        "zone": 1
    }, null, {
        "device": "sdb",
        "id": 2,
        "ip": "192.168.0.127",
        "meta": "",
        "port": 6202,
        "region": 1,
        "replication_ip": "192.168.0.127",
        "replication_port": 6202,
        "weight": 100.0,
        "zone": 1
    }, {
        "device": "sdb",
        "id": 3,
        "ip": "192.168.0.198",
        "meta": "",
        "port": 6202,
        "region": 1,
        "replication_ip": "192.168.0.198",
        "replication_port": 6202,
        "weight": 100.0,
        "zone": 1
    }, {
        "device": "sdb",
        "id": 4,
        "ip": "192.168.0.180",
        "meta": "",
        "port": 6202,
        "region": 1,
        "replication_ip": "192.168.0.180",
        "replication_port": 6202,
        "weight": 100.0,
        "zone": 1
    }, {
        "device": "sdb",
        "id": 5,
        "ip": "192.168.0.135",
        "meta": "",
        "port": 6202,
        "region": 1,
        "replication_ip": "192.168.0.135",
        "replication_port": 6202,
        "weight": 100.0,
        "zone": 1
    }],
    "part_shift": 22,
    "replica_count": 3
}
```


## building the ring {#building-the-ring}

<https://docs.openstack.org/swift/queens/overview%5Fring.html#building-the-ring>

```text
First the ring builder calculates the replicanths wanted at each tier in the ring’s topology based on weight.

Then the ring builder calculates the replicanths wanted at each tier in the ring’s topology based on dispersion.

Then the ring builder calculates the maximum deviation on a single device between its weighted replicanths and wanted replicanths.

Next we interpolate between the two replicanth values (weighted & wanted) at each tier using the specified overload (up to the maximum required overload). It’s a linear interpolation, similar to solving for a point on a line between two points - we calculate the slope across the max required overload and then calculate the intersection of the line with the desired overload. This becomes the target.

From the target we calculate the minimum and maximum number of replicas any partition may have in a tier. This becomes the replica-plan.

Finally, we calculate the number of partitions that should ideally be assigned to each device based the replica-plan.

On initial balance (i.e., the first time partitions are placed to generate a ring) we must assign each replica of each partition to the device that desires the most partitions excluding any devices that already have their maximum number of replicas of that partition assigned to some parent tier of that device’s failure domain.

When building a new ring based on an old ring, the desired number of partitions each device wants is recalculated from the current replica-plan. Next the partitions to be reassigned are gathered up. Any removed devices have all their assigned partitions unassigned and added to the gathered list. Any partition replicas that (due to the addition of new devices) can be spread out for better durability are unassigned and added to the gathered list. Any devices that have more partitions than they now desire have random partitions unassigned from them and added to the gathered list. Lastly, the gathered partitions are then reassigned to devices using a similar method as in the initial assignment described above.

Whenever a partition has a replica reassigned, the time of the reassignment is recorded. This is taken into account when gathering partitions to reassign so that no partition is moved twice in a configurable amount of time. This configurable amount of time is known internally to the RingBuilder class as min_part_hours. This restriction is ignored for replicas of partitions on devices that have been removed, as device removal should only happens on device failure and there’s no choice but to make a reassignment.

The above processes don’t always perfectly rebalance a ring due to the random nature of gathering partitions for reassignment. To help reach a more balanced ring, the rebalance process is repeated a fixed number of times until the replica-plan is fulfilled or unable to be fulfilled (indicating we probably can’t get perfect balance due to too many partitions recently moved).
```

翻译如下：

```text
首先，ring builder 根据weight计算 环的拓扑中每层所需的 replicanths。

然后，ring builder 根据dispersion计算 环形拓扑中每层所需的 replicanths。

然后，ring builder 计算单个设备在其weighted replicanths和wanted replicanths之间的最大偏差。

接下来，我们使用指定的过载（直到最大所需的过载）在每层的两个replicanth值（weighted & wanted）之间进行插值。这是一个线性插值，类似于求解两点之间的直线上的点 - 我们计算max required overload的斜率，然后计算该线与desired overload的交点。这成为目标。

从目标, 我们计算 在层中的 any partition 可能具有的最小和最大replicas数。这成为replica-plan。

Finally, we calculate the number of partitions that should ideally be assigned to each device based the replica-plan.

On initial balance（即，第一次放置分区以生成环）时，我们必须将每个分区的每个副本分配给需要最多分区的设备，不包括已分配给某些分区的最大分区副本数的设备该设备的故障域的父层。

当基于旧环而建立新环时，从当前replica-plan重新计算每个设备所需的分区数。接下来，将收集要重新分配的分区。任何已删除的设备都会将所有已分配的分区取消分配并添加到gathered list中。任何分区副本（由于添加了新设备）可以被分散以获得更好的持久性，这些副本是未分配的，会添加到gathered list中。任何具有比现在更多分区的设备都会从它们中取消分配随机分区，并添加到gathered list中。最后，使用与上述初始分配中类似的方法，将收集的分区重新分配给设备。

每当分区重新分配副本时，都会记录重新分配的时间。在收集分区以重新分配时会考虑这一点，以便在可配置的时间内不会移动分区两次。这个可配置的时间量在RingBuilder类内部称为min_part_hours。对于已删除的设备上的分区副本，将忽略此限制，因为设备删除应仅在设备故障时发生，并且除了进行重新分配之外别无选择。

由于收集分区用于重新分配的随机性质，上述过程并不总是能完美地rebalance a ring。为了帮助reach a more balanced ring，重新平衡过程重复固定次数，until replica-plan is fulfilled or unable to be fulfilled（表明, 由于最近移动的分区太多，我们可能无法获得 perfect balance）。
```

这里是对
