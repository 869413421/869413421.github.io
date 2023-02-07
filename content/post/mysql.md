---
title: "MySql（系统基础篇）"
date: 2023-02-07T15:52:40+08:00
draft: false
description: "MySql（系统基础篇）"
tags: ["MySQL","数据库"]
categories: ["MySQL","数据库"]
---

### 数据库三大范式是什么

* 第一范式：每个列不可拆分
* 第二范式：在第一范式基础上，所有非主键列完全依赖主键列，而不是主键的一部分
* 第三范式：在第二范式基础上，所有非主键列只依赖主键列，不依赖其他的非主键。

---

### MySql自带的权限表

* user权限表：记录允许连接到服务器的用户帐号信息，里面的权限是全局级的。
* db权限表：记录各个帐号在各个数据库上的操作权限。
* table_priv权限表：记录数据表级的操作权限。
* columns_priv权限表：记录数据列级的操作权限。
* host权限表：配合db权限表对给定主机上数据库级操作权限作更细致的控制。这个权限表不受GRANT和REVOKE语句的影响。

---

### MySql的binlog

binlog是MySql存储的二进制日志，用于记录用户操作Sql语句的信息

binlog具备三种模式

* STATMENT模式
  在STATMENT模式中用户每一条 `修改数据的SQL`都会记录到日志当中

优点：
不需要记录每一条SQL语句和每行的数据变化，减少磁盘的读写IO,减少数据库开销。

缺点：
在某些情况下会导致master-slave中的数据不一致(如sleep()函数， last_insert_id()，以及user-defined functions(udf)等会出现问题)

* ROW模式
  `不记录每一条SQL的上下文的信息`，只记录那一行被修改了，修改成什么样。

优点：不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发 `无法被正确复制的问题`。

缺点：`会产生大量的日志`，尤其是alter table的时候会让日志暴涨。

* 混合模式
  一般情况下使用STATMENT模式记录日志，在无法使用STATMENT模式时，切换为ROW模式。`MySql会根据执行的SQL来选择日志保存的方式`。

---

### binlog的设置

在MySQL配置文件my.cnf文件中的mysqld节中添加下面的配置文件：

[mysqld]

#设置日志格式

* binlog_format = mixed
* 设置日志路径，注意路经需要mysql用户有权限写
  log-bin = /data/mysql/logs/mysql-bin.log
* 设置binlog清理时间
  expire_logs_days = 7
* binlog每个日志文件大小
  max_binlog_size = 100m
* binlog缓存大小
  binlog_cache_size = 4m
* 最大binlog缓存大小
  max_binlog_cache_size = 512m

重启MySQL生效，如果不方便重启服务，也可以直接修改对应的变量即可。

### 引擎

常用的引擎

* InnoDB
  * InnoDB提供了ACID的事务支持，并且支持行级别锁，外键约束。
* MyIASM
  * MyIASM不支持事务，行级别锁。但是支持表级别锁
* MEMORY
  * MyIASM所有数据存储在内存中，读取速度快，但是安全性低

### MyIAM索引和InnoDB索引的区别

* InnoDB索引是聚簇索引，MyISAM索引是非聚簇索引。
* InnoDB索引的叶子节点上存储着索引和行数据，所以InnoDB的主键索引数据非常高效
* MyIASM索引的叶子节点上存储着行数据距的指向地址，需要再根据地址去进行查找
* InnoDB的非主键索引的叶子节点存储着主键和其他非主键索引的列数据

### InnoDB的四大特性

* 插入缓冲
  * 插入缓冲在非唯一索引非聚合索引才生效，当第一次插入时，MySQL会先检查buffer中是否包含索引页，如果有直接插入，如果没有先放置到buffer中，等到一定频率再合并操作。
  * 插入缓冲能讲多个操作合并成一个组，能减少IO消耗
  * 操作频率
    1.辅助索引页被读取到缓冲池中。正常的select先检查Insert Buffer是否有该非聚集索引页存在，若有则合并插入。

    2.辅助索引页没有可用空间。空间小于1/32页的大小，则会强制合并操作。

    3.Master Thread 每秒和每10秒的合并操作。
* 二次写
* 自适应hash索引
* 预读
