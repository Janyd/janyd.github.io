# MYSQL 执行计划详解


### 简介

![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919151542.png)

* **id** ，查询表格顺序，值越大优先级越高，id相同，执行顺序则从上到下
* **select_type**，查询类型，主要区分普通查询、联合查询、子查询
* **table**，查询所涉及到表
* **type**，访问类型，SQL查询优化的重要指标
* **possible_keys**，查询过程可能用到的索引
* **key**，查询实际上使用到的索引
* **key_len**，使用到的索引长度
* **ref**，谓词关联信息
* **rows**，根据表统计信息或者索引选用的情况，大致估算出找到所需记录需要读取的行数
* **filtered**，表示返回的结果行数占用需要读取行数的百分比，`filtered`值越大越好
* **extra**，特性使用

### ID

表格查询的顺序，降序查看，值越大优先级高，如相同则从上到下。

同时作为`table`进行引用，例如：`<union1,2>`，代表当前查询是由 id 为 1,2 表的并集查询结果



### SELECT_TYPE

查询类型主要有以下几种：

1. **SIMPLE**，简单查询方式，不使用 UNION 或子查询

   ```sql
   select * from user
   ```

   

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919153454.png)

2. **PRIMARY**，该表格位于最外层查询，通常与其他查询组合

3. **UNION**，`union`查询中的第二个或后面的 `select`

   ```sql
   select * from user union select * from user_2 ;
   ```

   

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919153907.png)

4. **SUBQUERY**，`select` 或 `where` 中包含有子查询

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919154427.png)

5. **DERVIED**，查询内使用内联视图

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919154811.png)

6. **MATERIALIZED**，子查询无话，表出现在非相关子查询中并且需要进行物化时，会出现`MATERIALIZED`关键词

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919155645.png)

7. **DEPENDENT_UNION**，子查询中的UNION操作，从UNION第二个及之后的所有 select 的select_type为 `DEPENDENT_UNION`，一般与 `DEPENDENT_SUBQUERY`结合应用

8. **DEPENDENT SUBQUERY**，子查询中第一个 select ，依赖于外部查询的结果集



### TYPE

`type` 是SQL查询优化的一个重要指标，其关键字性能排序：null > system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

1. **null**，不访问任何一个表，例如调用函数：`select now();`

2. **system**，表格有且仅有条数据时

3. **const**，主键或唯一索引的常量查询，最多只有一行符合匹配，一般用到主键或唯一索引定值查询

   ![image-20200919160935593](C:\Users\a3139\AppData\Roaming\Typora\typora-user-images\image-20200919160935593.png)

4. **eq_ref**，主键索引或非空唯一索引等值扫描，一般用在 join 查询过程，关联条件为主键或唯一索引，性能较好的join操作

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919161110.png)

5. **ref**，非及聚集索引（非主键非唯一索引）的常量查询

   ![](https://cdn.jsdelivr.net/gh/Janyd/blog-images/imgs/20200919161332.png)

6. **fulltext**，全文索引，查询过程中使用到了`fulltext`索引

7. **ref_or_null**，与 ref 查询类似，在 ref 查询基础之上，会加多一个null值查询条件

8. **index_merg**，当条件谓词使用多个索引的最左列并且谓词之间的连接是`or`情况下，会使用到索引联合查询

9. **unique_subquery**，eq_ref 查询一个分支，查询主键的子查询，例如：`value in (select id from table) where condition`

10. **index_subquery**，ref 查询一个分支，查询非聚集索引的子查询，例如`value in (select column from table) where condition`

11. **range**，使用到了范围查询，例如`>`、`>=`、`<`、`<=`、`BETWEEN`、`in`等等

12. **index**，索引树扫描，将扫描真个索引树，例如：`select count(*) from user` 或 `select age from user`

13. **all**，全表扫描，性能最差



### POSSIBLE_KEYS 、 KEY 与 KEY_LEN

`possible_keys`查询过程可能使用到的索引

`key` 实际用到的索引，如果为 NULL 则没使用索引

`key_len`使用到索引的长度，长度越短代表效率高



### REF

谓词关联信息，可能的值有 null、const 与 关联谓词的列名



### ROWS

根据表的统计信息及索引使用情况，大致估算出找出所需记录需要读取的行数，此参数很重要，直接反应的sql找了多少数据，在完成目的的情况下越少越好



### EXTRA

额外一些信息，这里会表示是否使用 文件排序 、临时表、覆盖索引、条件过滤等等

* **Using filesort**，对数据使用了一个外部文件内容进行排序，而不是按照表内的索引进行排序读取，性能较差需要优化
* **Using temporary**，使用到临时表来保存中间结果，对查询结果排序时使用了临时表，常见于 order by 或group by，有待优化，需要对order by 或 group by 后面的字段进行添加索引
* **Using index**，标识SQL使用了覆盖索引，避免了访问表的数据数据行
* **Using index condition** ，SQL查询命中了索引，但是不是所有数据都在索引上，还需要访问实际的行记录
* **Using where**，SQL操作使用了where过滤条件



### 总结

讲解了`MySQL`执行计划，如果遇到慢 SQL ，那么我们可以根据执行计划提供的信息对 SQL 进行优化，尤其是`type`提供的信息太重要了，描述了查询过程找到所需数据使用的扫描方式，同时建立正确的索引，也是非常重要的，所以经过了解执行计划，能够让我们怎么正确添加索引

### 参考链接

* [MySQL_执行计划详细说明](https://www.cnblogs.com/deityjian/p/11951436.html)
* [MySQL 语句执行过程详解](https://www.yuque.com/yinjianwei/vyrvkf/ri4ks7)
