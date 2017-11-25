---
title: mysql 索引
date: 2017-11-09 16:30
categories: 数据库
tags:
- MySQL
---

所有MySQL列类型都可以被索引，对相关列使用索引是提高SELECT操作性能的最佳途径。每种存储引擎对每个表至少支持16个索引，总索引长度至少为256字节。

<!-- more -->

* 语法
    
创建索引

``` 
create [unique|fulltext|spatial] index index_name [using idex_type] on table_name(idex_col_name,....)
```
    
删除索引
    
```
drop index index_name on table_name;
```
    
## 设计索引原则

- 搜索的索引列，不一定是所要选择的列。最适合索引的列是出现在where字句中的列，或连接字句中指定的列，而不是出现在select关键字后的选择列表中的列。
- 使用唯一索引。
- 使用短索引。
- 利用最左前缀。在创建一个n列的索引时，实际是创建了MySQL可利用的n个索引。多列索引可起几个索引的作用，因为可利用索引中最左边的列集来匹配行。这样的列集称为最左前缀。
- 不要过度索引。每个额外的索引都要占用额外的磁盘空间，并降低写操作的性能。在修改表的内容时，索引必须进行更新，有时可能需要重构，因此索引越多，所花的时间越长如果一个索引很少利用或者从不使用，那么会不必要地减缓表的修改速度。
- 对于InnoDB存储引擎的表，记录默认会按照一定的顺序保存，如果有明确定义的主键，则按照主键顺序保存。如果没有主键，但是有唯一索引，那么就是按照唯一索引的顺序保存。如果没有主键有没有唯一索引，那么表中会自动生成一个内部列， 按照这个列的顺序保存。按照主键或者内部列进行访问时最快的，所以InnoDB表尽量自己指定主键，当表中同时有几个列都是唯一的，都可以作为主键的时候，要选择最长作为访问条件的列作为主键，提高查询的效率。

## BTREE索引和HASH索引

两种不同的类型的索引各有其不同的适用范围，HASH索引有一些重要的特性需要在使用的时候特别注意。

- 只用于使用=或者<=>操作符的等式比较。
- 优化器不能使用HASH索引来加速ORDER BY操作。
- MySQL不能确定在两个值之间大约有多少行。如果将一个MyISAM表改为HASH索引的MEMORY表，会影响一些查询的执行效率。  
- 只能使用整个关键字来搜索一行。

而对BTREE索引，当使用`<`,`>`,`>=`,`<=`,`BETWEEN`,!=或者`<>`，或者LIKE 'pattern'操作符时，都可以使用相关列上的索引。

下列范围查询适合BTREE索引和HASH索引

```
select * from table_name where key_col = 1 or key_col in (15, 18, 20);
```

下列范围查询只适用于BTRR索引

```
select * from table_name where key_col > 1 and key_col < 10;

select * from table_name like 'ab%' or key_col between 'lisa' and 'simon';
```


