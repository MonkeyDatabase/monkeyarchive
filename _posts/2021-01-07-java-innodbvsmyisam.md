---
layout: post
title: Innodb vs MyISAM
excerpt: "Innodb是Mysql目前的默认存储引擎，MyISAM作为Mysql上一代默认存储引擎为什么被降为备用，本文将介绍Innode和MyISAM的区别和各自的适用场景。"
date:   2021-01-08 10:32:00
categories: [Web]
comments: true
---

## 学习笔记


### 1、事务

1. Innodb支持事务，事务安全
2. MyISAM不支持事务，非事务安全

### 2、锁

1. Innodb支持行级锁、页锁、表锁，其中行级锁是对数据库表中的某一行记录进行加锁
2. MyISAM仅支持表锁，`SELECT`、`UPDATE`、`DELETE`、`INSERT`都会加锁，且MyISAM引擎认为写操作比读操作优先，当写操作和读操作同时到达会优先写操作，当写操作很多时会造成读操作的饥饿

### 3、聚集索引

> 聚集索引指的是键值的顺序与数据行实际存储的顺序一致，一张表只能有一个聚集索引

1. MyISAM将表的所有行单独存储到B+树的外面，然后把各个行的引用存储到B+树的叶子节点中
2. Innodb将表中的各个行存储到B+树的叶子节点中，到达叶子节点后不需要再次寻址，使用聚集索引

### 4、行数

1. MyISAM在每个表中用一个变量记录了整个表的行数，在*SELECT COUNT(\*) FROM table_name*时可以直接返回该值，速度很快
2. Innodb则没有记录表的行数，如果执行*SELECT COUNT(\*) FROM table_name*时，会造成全表查询

### 5、外键

1. MyISAM不支持外键，不支持级联删除

2. Innodb支持外键

   > 目前不清楚外键的实现原理

### 6、适用场景

1. MyISAM：适用于以`SELECT` 和`INSERT`为主的应用，如博客系统、新闻门户网站
2. Innodb：适用于`UPDATE`和`DELETE`操作也比较高的应用，外键、事务、行锁是它的最大特点，适用于高并发系统，比如OA自动化办公系统

## Mysql引擎

| Engine             | Support | Comment                                                      | Transactions | XA   | Savepoints |
| ------------------ | ------- | ------------------------------------------------------------ | ------------ | ---- | ---------- |
| ARCHIVE            | YES     | Archive storage engine                                       | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                        | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                               | NULL         | NULL | NULL       |
| MyISAM             | YES     | MyISAM storage engine                                        | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                           | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys   | YES          | YES  | YES        |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables    | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                           | NO           | NO   | NO         |

* Engine：数据库引擎

  * 目前只使用过Innodb引擎和MyISAM引擎

    ```sql
    -- 通过该命令可以改变一个表采用的引擎为MyISAM
    alter table `table_name` engine=MyISAM
    -- 通过该命令可以改变一个表采用的引擎为Innodb
    alter table `table_name` engine=Innodb
    ```

  * mysql默认数据库中的performance_schema数据库，它里面采用的引擎是PERFORMANCE_SCHEMA引擎

    ```sql
    -- 使用该命令可以查看performance_schema的表状态，第二列就是采用的引擎
    show table status from performance_schema;
    ```

  * mysql默认数据库的mysql数据库，它里面有一个表general_log采用的引擎是CSV

    ```sql
    -- 使用该命令可以查看mysql数据库general_log的表状态，第二列就是采用的引擎
    show table status from mysql where name="general_log";
    ```

  * mysql默认数据库的information_schema数据库和sys数据库中的表状态里显示引擎为NULL

* Transaction：事务

* XA：一种分布式协议(具体内容未了解过)

* Savepoints

  ```sql
  -- some operation;
  savepoint x;
  -- some operation;
  rollback to x;
  -- some operation;
  ```

  