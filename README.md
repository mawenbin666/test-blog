###### 作者

马文斌

###### 时间

2022-07-07

###### 标签

清理大表 死元组 临时表

背景：近期收到智能运营管理平台的告警，发现

#### 统计大表和元组

```
SELECT
    table_name,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size,
    n_live_tup,
    n_dead_tup,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    seq_scan,
    idx_scan
   from (
 SELECT
        table_name,
        pg_table_size(table_name) AS table_size,
        pg_indexes_size(table_name) AS indexes_size,
        pg_total_relation_size(table_name) AS total_size,
        n_live_tup,
        n_dead_tup,
        n_tup_ins,
        n_tup_upd,
        n_tup_del,
        seq_scan,
        idx_scan
    FROM (
        SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name,
        t2.n_live_tup,
        t2.n_dead_tup,
        t2.n_tup_ins,
        t2.n_tup_upd,
        t2.n_tup_del,
        seq_scan,
        idx_scan
        FROM information_schema.tables as t1 join pg_stat_all_tables as t2 on t1.table_name =t2.relname 
    ) AS all_tables
    ORDER BY total_size desc)
    AS pretty_sizes
```

   效果如下：

![image-20220707161807096](https://s2.loli.net/2022/07/07/fmS18EahZ4bkFCT.png)



这样我们就知道哪些表占用空间比较大，比如死元组占用空间大就清理死原组，索引，临时表等都可能成为大表的因素之一，然后根据业务清理。



 索引使用为0的表：

```
select schemaname,relname,indexrelname,idx_scan from pg_stat_all_indexes where idx_scan =0 and schemaname not in ('pg_toast','pg_catalog')

```

