title: Mysql联合唯一索引和空值
date: 2016-02-14 10:05:39
tags:
- Mysql
---

## 问题

当Mysql中建立的联合索引， 只要索引中的某一列的值为空时(NULL)，即便其他的字段完全相同，也不会引起唯一索引冲突。

<!--more-->

## 原因

Mysql官方文档中有这样的解释

> A UNIQUE index creates a constraint such that all values in the index must be distinct. An error occurs if you try to add a new row with a key value that matches an existing row. This constraint does not apply to NULL values except for the BDB storage engine. For other engines, a UNIQUE index allows multiple NULL values for columns that can contain NULL.

唯一约束对NULL值不适用。原因可以这样解释： 比如我们有一个单列的唯一索引，既然实际会有空置的情况，那么这列一定不是`NOT NULL`的，如果唯一约束对空值也有起作用，就会导致仅有一行数据可以为空，这可能会和实际的业务需求想冲突的，所以通常Mysql的存储引擎的唯一索引对NULL值是不适用的。 这也就倒是联合唯一索引的情况下，只要某一列为空，就不会报唯一索引冲突。


## 解决

1. 使用BDB引擎。 这个方式牺牲太大了，通常不是个好选择。
2. 给会为空的列定义一个为空的特殊值来表示`NULL VALUE`。

或者可以重新审视下业务需求，某列可能为空，那么空值本身就不具有可比性，是不是就可以认为两行就是不一样的数据。

[mysql-compound-keys-and-null-values]: http://stackoverflow.com/questions/3086382/mysql-compound-keys-and-null-values