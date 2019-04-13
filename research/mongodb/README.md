# MongoDB从理论到实践

## MongoDB 简介

MongoDB是一个开源的分布式文档形数据库，文档是一个键值对组成的数据结构，类似JSON，字段的值可以是数组或者字典（可以理解为嵌套的文档），例如

![](https://docs.mongodb.com/manual/_images/crud-annotated-document.bakedsvg.svg)

MongoDB主打的特性包括

- 高性能
    - 支持嵌套的文档，从而减少了数据库的I/O
    - 支持在嵌套的文档或数组中创建索引
- 丰富的查询语言
    - [基本的增删改查](https://docs.mongodb.com/manual/crud/)
    - [数据聚合](https://docs.mongodb.com/manual/core/aggregation-pipeline/)
    - [文本搜索](https://docs.mongodb.com/manual/text-search/)
    - [地理空间数据查询](https://docs.mongodb.com/manual/tutorial/geospatial-tutorial/)
- 高可用
    - Primary故障后自动切换到Secondary
    - 提供数据冗余
- 水平扩展
    - 支持[Sharding](https://docs.mongodb.com/manual/sharding/#sharding-introduction)，把数据集分散到集群成员中
    - 支持基于[shard key](https://docs.mongodb.com/manual/reference/glossary/#term-shard-key)来创建[zone](https://docs.mongodb.com/manual/core/zone-sharding/#zone-sharding)从而尽可能的把相关较高的数据放在同样的zone中。
- 多种存储引擎支持

最后，作为一个NoSQL数据库，MongoDB不支持传统的ACID语意（4.0开始支持事务了），一致性需要应用层去保证。可能不适用于对一致性要求较高的业务。


## 为什么选择MongoDB

我们选用MongoDB的主要原因是上文提到的高性能：

- 支持嵌套的文档，从而减少了数据库的I/O
- 支持在嵌套的文档或数组中创建索引

在我们某个业务场景的性能测试中，MongoDB的有一项查询的效率大约是MariaDB的2.5倍。


## Replicate Set

为了避免单点故障，MongoDB提供了Replacate Set，它是经典的主备复制模式，客户端的读写I/O直接请求Primary，Primary *异步* 的把写I/O复制到所有Secondary。如果Primary宕机，其中一个Secondary节点会被选举为Primary，获得多数派投票的节点胜出。下文把Primary和Secondary统称为数据节点。

![](https://docs.mongodb.com/manual/_images/replica-set-read-write-operations-primary.bakedsvg.svg)

 MongoDB额外提供了一种Arbiter的节点，这种节点不参与数据存储和I/O请求，仅仅作为仲裁节点参与投票，很自然的，Arbiter节点无法成为Primary。

![](https://docs.mongodb.com/manual/_images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)

I/O的异步复制意味着这种可能性：Priamry在把I/O 写请求复制到Secondary前就宕机了，新选举出来的Primary缺失了一部分数据。MongoDB处理新老Primary数据不一致的方式是让老的Primary重新加入集群时Rollback这些没有复制的请求，参考[Rollbacks During Replica Set Failover](https://docs.mongodb.com/manual/core/replica-set-rollbacks/)。这个问题的另一个解决方式是*同步*模式，确认指定个数的Seoncdary已经写入这些数据后，Primary再回复客户端。MongoDB提供了一个[write concern](https://docs.mongodb.com/manual/reference/write-concern/)的配置，可以指定所需写入副本的个数，其中值得注意的是 `w: majority`，表示数据节点的多数派（不计仲裁节点）。

> The majority (M) is calculated as the majority of all voting members, but the write operation returns acknowledgement after propagating to M-number of data-bearing voting members (primary and secondaries with members[n].votes greater than 0).

通常集群系统不建议使用偶数节点，尤其是两个节点，双节点容易出现split brain的情况，MongoDB不存在这个问题， 因为双节点的MongoDB丢失任何一个节点都是不可用的。MongoDB建议的最小化部署是3个数据节点(PSS)，或者2个数据节点+1个arbiter(PSA)。然而，在PSA模式中，当write concern配置为 `w: majority`时，如果有任意一个数据节点故障，io是写不进去的（2个数据节点的多数派也是2)。

综上所述，我们决定采用3个数据节点的部署模式：

- 一方面三副本提供了更好的数据冗余
- 另一方面则是同样的部署模型可以简化流程和后续维护成本

最后，从容灾角度考虑

1. 把所有副本都放在一个数据中心是不可靠的
2. 如果把三副本放在不同的数据中心，当write concern 配置为 `w: majority`时可能带来较大的延迟
3. 我们选择了一个折中的方案：把两个副本放在数据中心A，把最后一个副本放在数据中心B，把数据中心A的两个副本priority配置为1，数据中心B的一个副本priority配置为0，避免后者当选为Primary。这样即便使用`w: majority`在正常情况下也能在合理的延迟内（从数据中心A的两副本）响应请求。这个方案的缺陷是数据中心A 整体故障时，集群将不可用，但至少我们有备份的数据可以用来快速恢复服务。

## 验证和授权

MongoDB令人诟病的一点是其默认配置不需要用户密码就能登陆，这导致了大量数据库泄露的案例。因此在生产环境设置合理的验证和授权是非常重要的，验证的目的明确登录用户的身份（换句话说，请证明你是你），授权指的是确定这个用户拥有权限访问什么内容。MongoDB的ACL比较奇怪，用户鉴权信息不是统一放在某个内部数据库，而是可以放在不同的数据库的，用户登陆时需要指定以哪个数据库来进行验证。

MongoDB需要验证的地方有个：

一是客户端到DB之间，需要防止恶意用户登陆，MongoDB 社区版提供了两种方案

- [SCRAM](https://docs.mongodb.com/manual/core/security-scram/) 这是MongoDB默认的验证方式，它是一种安全性较高的“Challenge/Reponse”验证机制
- [x.509证书验证](https://docs.mongodb.com/manual/core/security-x.509/#security-auth-x509) 这种机制需要启用SSL

二是MongoDB集群成员之间，需要防止恶意用户伪装成集群成员，MongoDB同样提供了两种方案

- [KeyFiles](https://docs.mongodb.com/manual/core/security-internal-authentication/#keyfiles) 它本质上是SCRAM，KeyFiles的内容就是集群成员的共享密钥。
- [x.509证书验证](https://docs.mongodb.com/manual/core/security-x.509/#security-auth-x509) 这种机制需要启用SSL。


如果是在公网上部署的系统，还要考虑信道的安全性，防止通讯被监听，业界通用的做法是TLS/SSL。我们的MongoDB集群用于内网，并且我厂的数据中心之间通过专线互联，信道的安全性不需要做太多的考虑，因此我们没有启用TLS/SSL，顺利成章的，用户验证我们使用了SCRAM，集群内部验证采用KeyFiles，在内网环境中已经能满足常规的安全策略要求。

## 存储和文件系统

数据库是典型的I/O密集型应用，对存储介质的要求较高，MongoDB建议使用RAID10和SSD。

我们使用的机型包含了4块2TB的nvme ssd，部署时使用LVM把4块NVME盘组成一个大的逻辑卷组，然后从中划分1TB mirror的逻辑卷（相当于Raid 1，实际占用2TB的空间）给MongoDB。这个方案的出发点主要是出于灵活性的考虑，剩余的空间可以按需分配，可以用来扩容，创建快照，又或者划分给其他业务。

我们已经有3副本做备份，原则上在单机存储层没有必要再做冗余，但是Raid1一方面通过空间换取了读取性能的提升；另一方面，磁盘是硬件中最容易损坏的设备之一，这种方案可以避免单块硬盘故障更换后重新同步数据的过程。

> 在Linux上创建RAID通常使用mdadm，然后在mdadm的基础上做lvm，这种方案是非常成熟的。我们采用的是另一种方案，直接把RAID 1(mirror)放到LVM上，一方面是出于灵活性的考虑，二是为了简化管理工具——用一个命令比两个命令简单。网上的讨论通常建议前者，然而我们决定吃一下螃蟹，假如后续遇到什么坑再来分享踩坑经验。由于项目进度比较紧张，我们缺乏足够的时间对两种方案进行详细的benchmark对比， 同时也没有找到最近几年在这方面的测试数据，这是后面可以再详细研究的一个方面。

文件系统方面MongoDB默认的WiredTiger存储引擎建议搭配XFS来使用（出于性能方面的考虑），我们保留了这一点。XFS在RHEL7中已经代替EXT4成为默认的文件系统，Redhat在[XFS in RHEL7 gonna be a good experience?](https://access.redhat.com/discussions/476563)中回应了这个决定的原因。

> 后续从我厂操作系统组了解到，目前xfs的bug非常多，hmm...

## 安装和部署流程

### 下载和安装

我厂的生产环境无法访问外网，因此我们直接从mongodb的安装源下载了这5个rpm安装包：

https://repo.mongodb.org/yum/redhat/7/mongodb-org/4.0/x86_64/RPMS/

```
root@:~/mongo# rpm -ivh *.rpm
warning: mongodb-org-4.0.6-1.el7.x86_64.rpm: Header V3 RSA/SHA1 Signature, key ID e52529d4: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mongodb-org-tools-4.0.6-1.el7    ################################# [ 20%]
   2:mongodb-org-shell-4.0.6-1.el7    ################################# [ 40%]
   3:mongodb-org-server-4.0.6-1.el7   ################################# [ 60%]
Created symlink from /etc/systemd/system/multi-user.target.wants/mongod.service to /usr/lib/systemd/system/mongod.service.
   4:mongodb-org-mongos-4.0.6-1.el7   ################################# [ 80%]
   5:mongodb-org-4.0.6-1.el7          ################################# [100%]
   ```

### 存储配置

启用[lvmated](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/logical_volume_manager_administration/metadatadaemon)

```
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
```

创建逻辑卷组vg0

```
pvcreate -f  /dev/nvme0n1
pvcreate -f  /dev/nvme1n1
pvcreate -f  /dev/nvme2n1
pvcreate -f  /dev/nvme3n1

vgcreate vg0 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/
```

以mirror（RAID 1）的方式创建1TB的逻辑卷data1

```
lvcreate -L 1T -m1 -n data1 vg0
```

可以看到这块逻辑卷使用了两块nvme上的空间

```
root@:~# lvs
  LV    VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data1 vg0  rwi-aor--- 1.00t                                    100.00
root@:~# pvs
  PV           VG   Fmt  Attr PSize PFree
  /dev/nvme0n1 vg0  lvm2 a--  1.64t 652.38g
  /dev/nvme1n1 vg0  lvm2 a--  1.64t 652.38g
  /dev/nvme2n1 vg0  lvm2 a--  1.64t   1.64t
  /dev/nvme3n1 vg0  lvm2 a--  1.64t   1.64t
```

创建XFS文件系统并挂载

```
mkfs.xfs /dev/vg0/data1
echo '/dev/vg0/data1 		/data1  		xfs	defaults,noac      	1 2' >> /etc/fstab
mount -a 
```

### OS参数配置

参考[Operating System Configuration](https://docs.mongodb.com/manual/administration/production-checklist-operations/#operating-system-configuration)

### MongoDB 配置

创建dbpath

```
mkdir -p /data/log/mongodb/
mkdir /data1/mongodb/

chown -R mongod:mongod /data1/mongodb
chown -R mongod:mongod /data/log/mongodb/
```

生成内部验证用的KeyFile，其实随便写点什么就好

```
openssl rand -base64 756 > /etc/mongodb-keyfile
chown mongod:mongod /etc/mongodb-keyfile
chmod 400 /etc/mongodb-keyfile
```

修改配置文件/etc/mongod.conf为
```
# cat /etc/mongod.conf | egrep -v '^#|^$'
systemLog:
  destination: file
  logAppend: true
  path: /data/log/mongodb/mongod.log
storage:
  dbPath: /data1/mongodb
  journal:
    enabled: true
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo
net:
  port: 27017
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
security:
  keyFile: /etc/mongodb-keyfile
  authorization: enabled
replication:
  replSetName: rs0
```

其中security指明了要启用鉴权，可是我们还没有配呢？mongod允许在启用鉴权后再创建管理员账户（仅允许一次）

### 启动MongoDB
 
 ```
 systemctl enable mongod
 systemctl start mongod
 ```

### MongoD 集群配置

首先通过mongo命令连接到任意一台机器
```
mongo --host <host> --port <port>
```


### 创建管理员账户

允许管理员账户读写任何数据库，权限保存在默认的admin数据库中

```
db.createUser(
  {
    user: "<user>",
    pwd: "<pwd>",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)
```

为管理员账户添加集群管理权限

```
db.grantRolesToUser(
   "root",
   [ "clusterManager" ]
)
```

管理员用户可以通过这个命令登陆

```
mongo --host <host> -p <port> -u root -p 
```

### 创建普通用户，用于业务

我们需要指定一个新的用于业务的数据库，不需要创建，直接use就行，然后创建一个包含读写权限的用户，应用程序使用这个用户来连接数据库。

```
use <dbname>
db.createUser(
  {
    user: "<user>",
    pwd: "<pwd>",
    roles: [ { role: "readWrite", db: "<dbname>" }]
  }
)
```

同理我们可以创建一个只读用户，用于生成报表之类的操作

```
db.createUser(
  {
    user: ""<user>"",
    pwd: "<pwd>",
    roles: [ { role: "read", db: "<dbname>" }]
  }
)
```

`db.getUser ('<user>')`可以查看一个用户拥有哪些权限

这些用户登陆时需要指定以哪个数据库作为验证源

```
mongo --host <host> --port <port> -u <user> -p  --authenticationDatabase <dbname>
```

### MongoDB 集群初始化

以管理员登陆mongo shell后执行

```
rs.initiate( {
   _id : "rs0",
   members: [
      { _id: 0, host: "<host1>:27017" },
      { _id: 1, host: "<host2>:27017" },
      { _id: 2, host: "<host3>:27017" }
   ]
})
```

`rs.conf()`可以检查集群配置，`rs.status()`可以检查集群状态

### 设置集群成员优先级

前面说过，我们有一个节点在别的数据中心，不希望它成为Primary，通过下面的方式可以调整成员的优先级，Priority 为0的成员不能成为Primary 

```
cfg = rs.conf()
cfg.members[0].priority = 10
cfg.members[1].priority = 10
cfg.members[2].priority = 0
rs.reconfig(cfg)
```

All Done

## 遗留问题

本文没有涉及这两个方面的内容，值得在后续进行研究：

- Sharding：如果以后数据量太大，或者并发数太高光靠Primary节点抗不住了，则需要考虑Sharding。
- 升级和回滚：比如当前部署的版本存在重大bug和安全漏洞的时候如何升级到新的版本；又或者更糟糕的，升级之后发现问题更多需要降回来。如何实现无缝升级，保证数据库文件在跨版本之间的可用性都是需要考虑的问题。