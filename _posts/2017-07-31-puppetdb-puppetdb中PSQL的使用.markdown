---
layout:     post
title:      "PuppetDB中PSQL的使用"
subtitle:   "puppetdb"
date:       2017-07-31 00:00:00
author:     "Luyuan"
header-img: "img/puppet4/puppetdb.png"
catalog: true
tags:
    - Python
---


## 1前言

## 1.1 puppetdb与PSQL
今早遇到一puppetdb问题，导致agent端无法运行，具体报错如下：
```ruby
Error: Could not retrieve catalog from remote server: Error 500 on SERVER: Server Error: A duplicate resource was found while collecting exported resources, with the type and title Sshkey[server-31.140.beishu.polex.io_rsa] on node server-31.225.gxdj.polex.io
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
```
从字面意思可以得出应该puppet在运行时资源导出重复定义导致的资源冲突问题，那么可以根据puppet原理，可以快速分析问题至puppetdb,同时puppetdb属于postgres数据库，突然发现对这个数据库不熟悉啊，不过还是有Google在。走你开始排查问题。先回顾一下简单SQL语句吧，下面主要是MySQL与PSQL对比，这样印象会更深刻。

## 1.2 MySQL与PSQL对比
```SQL
1.查看\?，可以显示出一些常见查询语句
psql: \?

2.列出所有的数据库
mysql: show databases
psql: \l或\list

3.切换数据库
mysql: use dbname
psql: \c dbname

4.列出当前数据库下的数据表
mysql: show tables
psql: \d

5.列出指定表的所有字段
mysql: show columns from table name
psql: \d tablename

6.查看指定表的基本情况
mysql: describe tablename
psql: \d+ tablename

7.退出登录
mysql: quit 或者\q
psql:\q

8.查看某个库中某个表的记录
select * from table_name limit 1;
```

## 1.3 开始操刀,直捣黄龙
```
1.3.1 切换用户
[root@server-20.140.beishu.polex.io ~ ]$ su postgres
1.3.2 进入数据库
[postgres@server-20.140.beishu.polex.io ~ ]$ psql
psql (9.2.15, server 9.4.5)
WARNING: psql version 9.2, server version 9.4.
         Some psql features might not work.
Type "help" for help.
postgres=# help
You are using psql, the command-line interface to PostgreSQL.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit

1.3.3 选择puppetdb数据库
postgres=# c\ puppetdb

1.3.4 显示puppetdb中所有的表
puppetdb=# \d
                     List of relations
 Schema |            Name            |   Type   |  Owner
--------+----------------------------+----------+----------
 public | catalog_resources          | table    | puppetdb
 public | catalogs                   | table    | puppetdb
 public | catalogs_id_seq            | sequence | puppetdb
 public | catalogs_transform_id_seq1 | sequence | puppetdb
 public | certname_id_seq            | sequence | puppetdb
 public | certnames                  | table    | puppetdb
 public | edges                      | table    | puppetdb
 public | environments               | table    | puppetdb
 public | environments_id_seq        | sequence | puppetdb
 public | fact_paths                 | table    | puppetdb
 public | fact_paths_id_seq          | sequence | puppetdb
 public | fact_values                | table    | puppetdb
 public | fact_values_id_seq         | sequence | puppetdb
 public | facts                      | table    | puppetdb
 public | factsets                   | table    | puppetdb
 public | factsets_id_seq            | sequence | puppetdb
 public | producers                  | table    | puppetdb
 public | producers_id_seq           | sequence | puppetdb
 public | report_statuses            | table    | puppetdb
 public | report_statuses_id_seq     | sequence | puppetdb
 public | reports                    | table    | puppetdb
 public | reports_id_seq             | sequence | puppetdb
 public | resource_events            | table    | puppetdb
 public | resource_params            | table    | puppetdb
 public | resource_params_cache      | table    | puppetdb
 public | schema_migrations          | table    | puppetdb
 public | value_types                | table    | puppetdb
(27 rows)


1.3.5 显示catalog_resources这个表详细信息
puppetdb=# \d+ catalog_resources
                     Table "public.catalog_resources"
   Column    |  Type   | Modifiers | Storage  | Stats target | Description
-------------+---------+-----------+----------+--------------+-------------
 resource    | bytea   | not null  | extended |              |
 tags        | text[]  | not null  | extended |              |
 type        | text    | not null  | extended |              |
 title       | text    | not null  | extended |              |
 exported    | boolean | not null  | plain    |              |
 file        | text    |           | extended |              |
 line        | integer |           | plain    |              |
 certname_id | bigint  | not null  | plain    |              |
Indexes:
    "catalog_resources_pkey1" PRIMARY KEY, btree (certname_id, type, title)
    "catalog_resources_encode_idx" btree (encode(resource, 'hex'::text))
    "catalog_resources_exported_idx" btree (exported) WHERE exported = true
    "catalog_resources_resource_idx" btree (resource)
    "catalog_resources_type_idx" btree (type)
    "catalog_resources_type_title_idx" btree (type, title)
Foreign-key constraints:
    "catalog_resources_certname_id_fkey" FOREIGN KEY (certname_id) REFERENCES certnames(id) ON DELETE CASCADE
    "catalog_resources_resource_fkey" FOREIGN KEY (resource) REFERENCES resource_params_cache(resource) ON DELETE CASCADE
Has OIDs: no

1.3.6 根据报错内容选择对应的字段
select * from catalog_resources where title like '%server-31%';
select * from catalog_resources where title like '%server-31%' and exported='t';
puppetdb=# select * from catalog_resources where title like '%server-31.140.beishu.polex.io%' and exported='t';
                  resource                  |                                                                                                       tags
                                                                                                 |  type  |                 title                 | exported |
                                      file                                      | line | certname_id
--------------------------------------------+-----------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------+--------+---------------------------------------+----------+
--------------------------------------------------------------------------------+------+-------------
 \xc7f222959a0a21dcce465987f8fcdc2a8957e913 | {sunfire::base,server,class,sunfire,base,server-31.140.beishu.polex.io_ecdsa,__node_regexp__server-31-3.140.beis
hu.polex.io,ssh::server,sshkey,ssh::hostkeys,hostkeys,ssh,sunfire::controller,controller,node}   | Sshkey | server-31.140.beishu.polex.io_ecdsa   | t        |
 /etc/puppetlabs/code/environments/production/modules/ssh/manifests/hostkeys.pp |   37 |           7
 \x794b3d643fbe689292c1ce317c9c93c2fc44ee8b | {sunfire::base,server,class,server-31.140.beishu.polex.io_ed25519,sunfire,base,__node_regexp__server-31-3.140.be
ishu.polex.io,ssh::server,sshkey,ssh::hostkeys,hostkeys,ssh,sunfire::controller,controller,node} | Sshkey | server-31.140.beishu.polex.io_ed25519 | t        |
 /etc/puppetlabs/code/environments/production/modules/ssh/manifests/hostkeys.pp |   50 |           7
 \x5eb759b65d5ea6ab924f35f5fce225ae8206905f | {sunfire::base,server,class,sunfire,base,__node_regexp__server-31-3.140.beishu.polex.io,ssh::server,sshkey,ssh::
hostkeys,hostkeys,server-31.140.beishu.polex.io_dsa,ssh,sunfire::controller,controller,node}     | Sshkey | server-31.140.beishu.polex.io_dsa     | t        |
 /etc/puppetlabs/code/environments/production/modules/ssh/manifests/hostkeys.pp |   19 |           7
 \xe9ef737ad0a514e9a3e16270f5f2fa2222af64ee | {server-31.140.beishu.polex.io_rsa,sunfire::base,server,class,sunfire,base,__node_regexp__server-31-3.140.beishu
.polex.io,ssh::server,sshkey,ssh::hostkeys,hostkeys,ssh,sunfire::controller,controller,node}     | Sshkey | server-31.140.beishu.polex.io_rsa     | t        |
 /etc/puppetlabs/code/environments/production/modules/ssh/manifests/hostkeys.pp |   25 |           7
 \x7cb5c92f754a3fe91ed10ec343dd0182b8833e95 | {sunfire::base,server,class,sunfire,base,server-31.140.beishu.polex.io_ecdsa,__node_regexp__server-31-3.140.beis
hu.polex.io,ssh::server,sshkey,ssh::hostkeys,hostkeys,ssh,sunfire::controller,controller,node}   | Sshkey | server-31.140.beishu.polex.io_ecdsa   | t        |
 /etc/puppetlabs/code/environments/production/modules/ssh/manifests/hostkeys.pp |   37 |           4
 \x00114e25ba8f585c34107a1b907a5c0b0ff24515 | {sunfire::base,server,class,server-31.140.beishu.polex.io_ed25519,sunfire,base,__node_regexp__server-31-3.140.be
ishu.polex.io,ssh::server,sshkey,ssh::hostkeys,hostkeys,ssh,sunfire::controller,controller,node} | Sshkey | server-31.140.beishu.polex.io_ed25519 | t        |
 /etc/puppetlabs/code/environments/production/modules/ssh/manifests/hostkeys.pp |   50 |           4
(6 rows)


1.3.7 根据报错内容删除对应的冲突资源
puppetdb=# delete from catalog_resources where title like '%server-31.140.beishu.polex.io%' and exported='t';
DELETE 8
puppetdb=# delete from catalog_resources where title like '%server-32.140.beishu.polex.io%' and exported='t';
DELETE 8
puppetdb=# delete from catalog_resources where title like '%server-33.140.beishu.polex.io%' and exported='t';
DELETE 8
```
## 1.8 测试agent端是否可用
```ruby
puppet agent -t -debug
Info: Caching catalog for server-31.225.gxdj.polex.io
Warning: Passing port to firewall is deprecated and will be removed. Use dport and/or sport instead.
Debug: /Firewall[102 allow memcached access]: [validate]
Debug: Puppet::Type::Rabbitmq_user::ProviderRabbitmqctl: file rabbitmqctl does not exist
Debug: Puppet::Type::Rabbitmq_user_permissions::ProviderRabbitmqctl: file rabbitmqctl does not exist
Debug: Puppet::Type::Rabbitmq_vhost::ProviderRabbitmqctl: file rabbitmqctl does not exist
Debug: Puppet::Type::Rabbitmq_user::ProviderRabbitmqctl: file rabbitmqctl does not exist
Debug: Puppet::Type::Rabbitmq_user_permissions::ProviderRabbitmqctl: file rabbitmqctl does not exist
^CNotice: Caught INT; exiting
^C^C

完美运行
```
## 1.9总结
通过简单PSQL操作回顾来解决问题，需要了解Puppet原理和Puppetdb是如何存储resource资源，这样我们可以简单快速办法去定位问题。

—— Luyuan 后记于 2017.07.31
