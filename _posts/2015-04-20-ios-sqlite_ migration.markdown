---
layout: post
cover: 'assets/images/cover11.jpeg'
navigation: True
title: "iOS SQLite 数据库迁移"
date: 2015-04-20 21:06:15 +0800
tags: iOS
subclass: 'post tag-speeches'
logo: 'assets/images/ghost.png'
author: yiming
categories: yiming
description: "最近不得不考虑关于数据库迁移的问题，原先用了种很不好的处理方式（每次版本升级就删除本地数据库，太傻），于是开始考虑下如何迁移数据库。"
---
最近不得不考虑关于数据库迁移的问题，原先用了种很不好的处理方式（每次版本升级就删除本地数据库，太傻），于是开始考虑下如何迁移数据库。

项目使用的 [FMDB](https://github.com/ccgus/fmdb)。在 [FMDB](https://github.com/ccgus/fmdb) 介绍页面，发现了 [FMDBMigrationManager](https://github.com/layerhq/FMDBMigrationManager) ，大喜。

看了半天文档，捣鼓了半天才弄出来，一步步整理下。

## 安装 FMDBMigrationManager

Podfile 文件：

``` C
platform :ios, "7.0"

pod 'FMDB'
pod 'FMDBMigrationManager'
```
使用 ``pod install`` 命令安装

## FMDBMigrationManager 创建数据库 

``` C
FMDBMigrationManager *manager = [FMDBMigrationManager managerWithDatabaseAtPath:[YMDatabaseHelper databasePath]  migrationsBundle:[NSBundle mainBundle]];
```

其中``[YMDatabaseHelper databasePath]``是数据库路径

## 创建迁移表  

创建的迁移表名称为：``schema_migrations``

``` C
[manager createMigrationsTable:&error];
```

## 创建 .sql 文件

该文件用来存储每次升级使用的 SQL 语句。

``FMDBMigrationManager`` 建议我们使用时间戳来作为版本号，使用下面的命令生成一个文件：

```
touch "`ruby -e "puts Time.now.strftime('%Y%m%d%H%M%S%3N').to_i"`"_CreateMyAwesomeTable.sql
```
我生成的文件名为：``20150420170044940_CreateMyAwesomeTable.sql``,其中``20150420170044940`` 为迁移的版本号标识。

我们在 ``20150420170044940_CreateMyAwesomeTable.sql`` 文件中创建一个用户表，写入：

``` SQL
CREATE TABLE User(
id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
name TEXT
);
```

## 迁移函数  

``` C
FMDBMigrationManager *manager = [FMDBMigrationManager managerWithDatabaseAtPath:[YMDatabaseHelper databasePath]  migrationsBundle:[NSBundle mainBundle]];

BOOL resultState = NO;
NSError *error = nil;
if (!manager.hasMigrationsTable) {
resultState = [manager createMigrationsTable:&error];
}

resultState = [manager migrateDatabaseToVersion:UINT64_MAX progress:nil error:&error];//迁移函数

NSLog(@"Has `schema_migrations` table?: %@", manager.hasMigrationsTable ? @"YES" : @"NO");
NSLog(@"Origin Version: %llu", manager.originVersion);
NSLog(@"Current version: %llu", manager.currentVersion);
NSLog(@"All migrations: %@", manager.migrations);
NSLog(@"Applied versions: %@", manager.appliedVersions);
NSLog(@"Pending versions: %@", manager.pendingVersions);
```

``UINT64_MAX`` 表示把数据库迁移到最大的版本

运行项目，打印出如下内容：

```
2015-04-20 20:50:19.033 YMFMDatabase[12654:1326201] Has `schema_migrations` table?: YES
2015-04-20 20:50:19.036 YMFMDatabase[12654:1326201] Origin Version: 20150420170044940
2015-04-20 20:50:19.036 YMFMDatabase[12654:1326201] Current version: 20150420170044940
2015-04-20 20:50:19.037 YMFMDatabase[12654:1326201] All migrations: (
"<FMDBFileMigration: 0x17003b4c0>"
)
2015-04-20 20:50:19.037 YMFMDatabase[12654:1326201] Applied versions: (
20150420170044940
)
2015-04-20 20:50:19.038 YMFMDatabase[12654:1326201] Pending versions: (
)
```

用 iFunBox 查看下是不是创建了一个 User 表，里面含有 ``id`` 、``name`` 字段。以及 ``FMDBMigrationManager`` 生成的 ``schema_migrations`` 表。  

## 创建第 2 个 .sql 文件

先用上方命令: ``touch "`ruby -e "puts Time.now.strftime('%Y%m%d%H%M%S%3N').to_i"`"_CreateMyAwesomeTable.sql`` 生成,我生成的是： ``20150420170557221_CreateMyAwesomeTable.sql`` 。

第二个 sql 文件，里面创建一个新表名字为 Grouping ，为原先的 User 表添加邮箱字段：``email`` 。.sql 文件修改如下：

``` SQL
CREATE TABLE Grouping(
id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
name TEXT
);

ALTER TABLE User ADD email TEXT;
```

## 创建第 3 个 .sql 文件

生成同上，文件内容是为 Grouping 表添加备注字段 ``remark`` ：

文件内容：  

``` SQL
ALTER TABLE Grouping ADD remark TEXT;
```

OK ，直接运行项目，看看是不是创建了 Grouping 表，里面含有 ``id``, ``name`` , ``remark`` 字段，以及 User 表里面是不是添加了 ``email`` 字段。


## 遇到的问题

中间我自己做 Demo 时，试图删除表中的某列，比如删除 User 中的 ``name`` 列，但是不能成功，Google 了下发现[答案](http://stackoverflow.com/questions/8442147/how-to-delete-or-add-column-in-sqlite)。

>SQLite supports a limited subset of ALTER TABLE. The ALTER TABLE command in SQLite allows the user to rename a table or to add a new column to an existing table. It is not possible to rename a column, remove a column, or add or remove constraints from a table.

就是说 SQLite 对 ``ALERT TABLE`` 命令受限制，SQLite 中的 ``ALERT TABLE`` 命令只能允许用户重命名表或者添加新列，不能重命名列或者删除列或者删除约束。


