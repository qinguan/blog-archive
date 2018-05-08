---
title: SQL语句中的NULL判断
date: 2018-04-26 19:22:16
tags: 
    - SQL
---

今天在写SQL过滤条件时，有个URL判空逻辑，想当然的写成了如下格式:
```SQL
    select * from record where url != null    
```
然而，明明有符合条件的数据，但查询结果并没有出现。问题在哪呢？
检索了一下，发现SQL语句中针对NULL的处理逻辑有些特殊。

```
NULL表示一个未知的值，不能与任何类型的数据进行比较。如果进行了比较，则比较结果依然是NULL，而不是True或者False。
```

那如何判断呢？答案是，使用is NULL 或者 is not  NULL。
上面的SQL语句改成如下即可。
```SQL
    select * from record where url is not null   
```

MySQL手册还给了几个比较典型的case，有助于进一步理解:
```SQL
    mysql> SELECT 1 IS NULL, 1 IS NOT NULL;
    +-----------+---------------+
    | 1 IS NULL | 1 IS NOT NULL |
    +-----------+---------------+
    |         0 |             1 |
    +-----------+---------------+

    mysql> SELECT 1 = NULL, 1 <> NULL, 1 < NULL, 1 > NULL;
    +----------+-----------+----------+----------+
    | 1 = NULL | 1 <> NULL | 1 < NULL | 1 > NULL |
    +----------+-----------+----------+----------+
    |     NULL |      NULL |     NULL |     NULL |
    +----------+-----------+----------+----------+

    mysql> SELECT 0 IS NULL, 0 IS NOT NULL, '' IS NULL, '' IS NOT NULL;
    +-----------+---------------+------------+----------------+
    | 0 IS NULL | 0 IS NOT NULL | '' IS NULL | '' IS NOT NULL |
    +-----------+---------------+------------+----------------+
    |         0 |             1 |          0 |              1 |
    +-----------+---------------+------------+----------------+
```

完整说明请参考[MySQL手册-Working with NULL Values](https://dev.mysql.com/doc/refman/8.0/en/working-with-null.html)。
