---
title: how to back up and restore a mysql database
date: 2017-10-24 11:11:00
categories: 数据库
tags:
- MySQL
---

### MySQL dump

使用mysqldump command,这条命令连接数据库，然后创建一个SQL dump 文件。这个dump文件包含了重新建库的所需SQL语句。正确的语法是这样的:

`$ mysqldump --opt -u [uname] -p[pass] [dbname] > [backupfile.sql];`

<!--more-->

* uname: 数据库连接用户名
* pass: 数据库连接密码
* dbname: 要备份的库
* backupfile.sql: 备份的文件名
* --opt: mysqldump option


例子：

`$ mysqldump -u root -p Tutorials > tut_backup.sql;`

当然你也可以指定备份的表

`$ mysqldump -u root -p Tutorials php_tutorials asp_tutorials > tut_backup.sql;`

有些情况下，可能需要一次性dump多个数据库，在这种情况下，你可以使用 '--database' option，后面跟上你要dump的数据库列表，数据库名之间使用空格隔开:

`$ mysqldump -u root -p --database Tutorials Articles Comments > content_backup.sql;` 

如果一次性dump数据库服务器中的所有库，你可以使用 '--all-databases' option

`$ mysqldump -u root -p --all-databases > alldb_backup.sql;`

mysqldump命令也有有些其他的options:

* --add-drop-table: 声明在创建表之前添加一条删除表的 SQL 语句
* --no-data: 只 dump 数据结构，不 dump 内容
* add-locks: 在 dump 文件中添加lock table 和 unlock table 语句

mysqldump的优缺点:

* 优点
易用性，能解决表锁定问题

* 缺点
缺点就是锁表语句，如果表很大，mysqldump会在很长一段时间组织用户访问

### 备份 MySQL 数据库到压缩文件
如果你的库非常大，你可能想压缩输出你的mysqldump。使用下面的mysql备份命令管道输出gzip，就能获取一个压缩文件

`$ mysqldump -u [uname] -p[pass] [dbname] | gzip -9 > [backupfile.sql.gz];`

解压命令

`$ gunzip [backupfile.sql.gz]`

### 恢复MySQL数据库

上面介绍了备份 Tutorials 库到tut_backup.sql, 为了重新创建Tutorial数据库，需要
下面两个步骤:

* 在目标机器上创建一个合适的数据库名称
* 使用mysql命令加载备份文件

`$ mysql -u [uname] -p[pass] [db_to_restore] < [backupfile.sql];`

看看你是怎样 backupfile.sql 恢复Tutorials库的

`$ mysql -u root -p Tutorials < backupfile.sql;`

从压缩文件恢复

`gunzip < [backupfile.sql.gizp] | mysql -u [uname] -p [pass] [dbname];`

如果要恢复的库已经存在了，你需要使用mysqlimport命令

`mysqlimport -u [uname] -p[pass] [dbname] [backupfile.sql];`

