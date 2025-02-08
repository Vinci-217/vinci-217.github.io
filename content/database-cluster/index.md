---
title: 一文归纳数据库集群技术要点
date: 2025-02-07T19:52:18+08:00
tags: ["MySQL", "Redis", "Cluster"]
series: []
featured: true
---

数据库是服务端开发离不开的中间件，为了提高大型项目中数据库的可用性，常常通过集群的方式部署数据库。本文将从数据库集群的技术要点出发，介绍基于 MySQL 和 Redis 的数据库集群方案。

<!--more-->

## MySQL

MySQL 的集群出现主要是为了应对高并发读写和数据库宕机的场景。针对这样的场景，MySQL 采用了多个服务集群部署、读写分离等策略来应对。MySQL 集群的方式有很多种，目的都是为了提高其可用性。

![MySQL 集群架构](image/image.png)

### 集群模式

#### MySQL Replication

MySQL Replication 最基本的 MySQL 集群功能，基于一主多从的架构，主库负责写数据，从库负责读数据。主库会将数据变更记录在 binlog 中，从库通过读取主库的 binlog 来获取主库的最新数据，相当于主库的 sql 语句在从库上又执行了一遍。

#### MySQL Fabirc

MySQL Fabric 是在 MySQL Replication 的基础上，增加的故障检测与转移、自动数据分片的功能。但是依然是基于一主多从的架构，主库负责写数据，从库负责读数据。MySQL Fabric 只有一个主节点，但是当主节点挂掉以后，会从从节点中选一个来当主节点。

#### MySQL Cluster

MySQL Cluster 是一种多主多从的架构，也是由 MySQL 官方提供。他的高可用、负载均衡、伸缩性都很优秀，但是架构模式和原理很复杂，并且只能使用 NDB 存储引擎而不是 InnoDB 存储引擎（比如在事务隔离级别上只支持 Read Committed）。

### 主从同步

MySQL 的集群之间的数据同步是基于 binlog 的。binlog 是 MySQL 服务器的二进制日志，记录了对数据库的修改，包括增删改操作。通过 binlog，可以将数据同步到其他的 MySQL 服务器，实现数据库集群的数据一致性。

binlog 有三种格式，一种是 statement，一种是 row，还有一种是 mixed。statement 格式的 binlog 记录的是 SQL 语句的原始文本，row 格式的 binlog 记录的是每行数据的修改，mixed 格式的 binlog 既记录 SQL 语句，又记录每行数据的修改。

假如我们执行一个删除的 SQL，`delete from table1 where id > 100 limit 1`，由于 limit 这个命令，可能导致从库具体的这个 limit 1 和主库的 limit 1 不是同一行数据，所以造成误删的风险，那么 row 格式的 binlog 就应运而生了。但是由于每次记录 row 类型的 binlog 对内存开销太大，所以就有了 mixed 格式的 binlog——既记录 SQL 语句，又记录每行数据的修改，做了两者之间的权衡。

不同的集群模式主从同步的方式也不太一样，但大致流程相似：

1. 主库接收到客户端的更新请求，执行更新操作并写入 binlog
2. 从库在主从之间简历长连接
3. 主库的 dump_tread 从本地读取 binlog 给从库
4. 从库获取到主库的 binlog 后存储到本地，成为 relay log
5. 从库的 sql_thread 读取 relay log，解析出具体的 sql 语句，执行 sql 语句

### 主从延迟

### 读写分离

## Redis

## 参考文献

- [多图文，详细介绍 mysql 各个集群方案](https://www.cnblogs.com/lgx211/p/12456859.html)
- [MySQL 中常见的几种高可用架构部署方案](https://www.cnblogs.com/ricklz/p/17335755.html)
