---
title: SQL 编写规范
date: 2020-12-29 18:00:16
tags:
- Mysql
- SQL
categories: 
- 数据库
cover: /photo/u=1366396321,140410649&fm=26&gp=0.jpg

---
#### SQL 编写规范

###### 注意事项
1. UPDATE、DELETE 操作不使用 LIMIT，必须走 WHERE 精准匹配，LIMIT 是随机的，此类操作会导致数据出错。

2. 禁止使用INSERT INTO t_xxx VALUES(xxx)，必须显式指定插入的列属性，避免表结构变动导致数据出错。

4. SQL 语句中最常见的导致索引失效的情况需注意：隐式类型转换，如索引 a 的类型是 varchar，SQL 语句写成 where a = 1; varchar 变成了int。

5. 对索引列进行数学计算和函数等操作，例如，使用函数对日期列进行格式化处理。
join 列字符集不统一。

6. 多列排序顺序不一致问题，如索引是 (a,b)，SQL 语句是 order by a b desclike。
模糊查询使用的时候对于字符型xxx%形式可以走到一些索引，其他情况都走不到索引。
7. 使用了负方向查询（not，!=，not in 等）。==

###### 建议事项
1. 按需索取，拒绝select *，规避以下问题：无法索引覆盖，回表操作，增加 I/O。
额外的内存负担，大量冷数据灌入innodb_buffer_pool_size，降低查询命中率。
额外的网络传输开销。
2. 尽量避免使用大事务，建议大事务拆小事务，规避主从延迟。
业务代码中事务及时提交，避免产生没必要的锁等待。
3. 少用多表 join，大表禁止 join，两张表 join 必须让小表做驱动表，join 列必须字符集一致并且都建有索引。
4. LIMIT 分页优化，LIMIT 80000，10这种操作是取出80010条记录，再返回后10条，数据库压力很大，推荐先确认首记录的位置再分页，例如SELECT * FROM test WHERE id = ( SELECT sql_no_cache id FROM test order by id LIMIT 80000,1 ) LIMIT 10 ;。
5. 避免多层子查询嵌套的 SQL 语句，MySQL 5.5 之前的查询优化器会把 in 改成 exists，会导致索引失效，若外表很大则性能会很差。