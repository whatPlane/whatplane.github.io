---
layout:     post                    # 使用的布局（不需要改）
title:      mysql-insert-update-delete              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - mysql
---

#### insert
这里是插入一条记录
基本语法：
```
INSERT INTO <表名> (字段1, 字段2, ...) VALUES (值1, 值2, ...);
```
注意如果是自动增量就不需要在字段名里面特别列出来了。
例子。添加武松和西门庆到列表中,都是22岁
```
MariaDB [menagerie]> select * from tmp;
+----+---------+------+
| id | name    | age  |
+----+---------+------+
|  6 | david_x |   33 |
|  7 | cola_x  |   28 |
|  8 | pang_x  | NULL |
|  9 | NULL    | NULL |
+----+---------+------+
4 rows in set (0.00 sec)

MariaDB [menagerie]> insert into tmp(name,age)values('wusong',22);
Query OK, 1 row affected (0.00 sec)
MariaDB [menagerie]> insert into tmp(name,age)values('ximenqing',22);
Query OK, 1 row affected (0.00 sec)

MariaDB [menagerie]> select * from tmp;
+----+-----------+------+
| id | name      | age  |
+----+-----------+------+
|  6 | david_x   |   33 |
|  7 | cola_x    |   28 |
|  8 | pang_x    | NULL |
|  9 | NULL      | NULL |
| 10 | wusong    |   22 |
| 11 | ximenqing |   22 |
+----+-----------+------+
6 rows in set (0.00 sec)

```


#### update
基本语法：
```
UPDATE <表名> SET 字段1=值1, 字段2=值2, ... WHERE ...;
```
注意如果没有设置where会修改所有字段值。所有修改前最好用where确认下。

- 更新一条记录


把不及格的同学的分数改成60分。= =
```
MariaDB [test1]> select * from students;
+----+----------+--------+--------+-------+
| id | class_id | name   | gender | score |
+----+----------+--------+--------+-------+
|  1 |        1 | 小明   | M      |    90 |
|  2 |        1 | 小红   | F      |    95 |
|  3 |        1 | 小军   | M      |    88 |
|  4 |        1 | 小米   | F      |    73 |
|  5 |        2 | 小白   | F      |    81 |
|  6 |        2 | 小兵   | M      |    55 | -- 小兵
|  7 |        2 | 小林   | M      |    85 |
|  8 |        3 | 小新   | F      |    91 |
|  9 |        3 | 小王   | M      |    89 |
| 10 |        3 | 小丽   | F      |    85 |
| 11 |        5 | 新生   | M      |    88 |
+----+----------+--------+--------+-------+
11 rows in set (0.00 sec)

MariaDB [test1]> update students set score=60 where score<60;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [test1]> select * from students;
+----+----------+--------+--------+-------+
| id | class_id | name   | gender | score |
+----+----------+--------+--------+-------+
|  1 |        1 | 小明   | M      |    90 |
|  2 |        1 | 小红   | F      |    95 |
|  3 |        1 | 小军   | M      |    88 |
|  4 |        1 | 小米   | F      |    73 |
|  5 |        2 | 小白   | F      |    81 |
|  6 |        2 | 小兵   | M      |    60 | -- 小兵
|  7 |        2 | 小林   | M      |    85 |
|  8 |        3 | 小新   | F      |    91 |
|  9 |        3 | 小王   | M      |    89 |
| 10 |        3 | 小丽   | F      |    85 |
| 11 |        5 | 新生   | M      |    88 |
+----+----------+--------+--------+-------+
11 rows in set (0.00 sec)

```
- 更新多条记录

把没有80分的都加10分。**这里主要验证查询数据并且修改数据**
```
MariaDB [test1]> select * from students;
+----+----------+--------+--------+-------+
| id | class_id | name   | gender | score |
+----+----------+--------+--------+-------+
|  1 |        1 | 小明   | M      |    90 |
|  2 |        1 | 小红   | F      |    95 |
|  3 |        1 | 小军   | M      |    88 |
|  4 |        1 | 小米   | F      |    73 | ---
|  5 |        2 | 小白   | F      |    81 |
|  6 |        2 | 小兵   | M      |    55 | ---
|  7 |        2 | 小林   | M      |    85 |
|  8 |        3 | 小新   | F      |    91 |
|  9 |        3 | 小王   | M      |    89 |
| 10 |        3 | 小丽   | F      |    85 |
| 11 |        5 | 新生   | M      |    88 |
+----+----------+--------+--------+-------+
11 rows in set (0.00 sec)

MariaDB [test1]> update students set score=80 where score<80;
Query OK, 2 rows affected (0.01 sec)
Rows matched: 2  Changed: 2  Warnings: 0

MariaDB [test1]> select * from students;
+----+----------+--------+--------+-------+
| id | class_id | name   | gender | score |
+----+----------+--------+--------+-------+
|  1 |        1 | 小明   | M      |    90 |
|  2 |        1 | 小红   | F      |    95 |
|  3 |        1 | 小军   | M      |    88 |
|  4 |        1 | 小米   | F      |    83 | ---
|  5 |        2 | 小白   | F      |    81 |
|  6 |        2 | 小兵   | M      |    65 | ---
|  7 |        2 | 小林   | M      |    85 |
|  8 |        3 | 小新   | F      |    91 |
|  9 |        3 | 小王   | M      |    89 |
| 10 |        3 | 小丽   | F      |    85 |
| 11 |        5 | 新生   | M      |    88 |
+----+----------+--------+--------+-------+
11 rows in set (0.00 sec)


```
#### delete
这里是删除记录
基本语法：
```
DELETE FROM <表名> WHERE ...;
```

例子删除 西门庆和武松
```
MariaDB [menagerie]> select * from tmp;
+----+-----------+------+
| id | name      | age  |
+----+-----------+------+
|  6 | david_x   |   33 |
|  7 | cola_x    |   28 |
|  8 | pang_x    | NULL |
|  9 | NULL      | NULL |
| 10 | wusong    |   22 |
| 11 | ximenqing |   22 |
+----+-----------+------+
6 rows in set (0.00 sec)

MariaDB [menagerie]> delete from tmp where age=22;
Query OK, 2 rows affected (0.00 sec)

MariaDB [menagerie]> select * from tmp;
+----+---------+------+
| id | name    | age  |
+----+---------+------+
|  6 | david_x |   33 |
|  7 | cola_x  |   28 |
|  8 | pang_x  | NULL |
|  9 | NULL    | NULL |
+----+---------+------+
4 rows in set (0.00 sec)

```
### 结束