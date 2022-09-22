# Scada-系统

一开始是没有打开慢查询的配置的。

![image-20220920134714166](https://raw.githubusercontent.com/mawenbin666/blog-img/main/202209201347509.png)



# EMS系统

分析服务器ip：10.8.39.66 

目前慢查询阈值为15秒，超过15秒则记录

```
localhost 14:05:05 [mes_prd]> show variables like '%query%';
+------------------------------+--------------------------------+
| Variable_name                | Value                          |
+------------------------------+--------------------------------+
| binlog_rows_query_log_events | ON                             |
| ft_query_expansion_limit     | 20                             |
| have_query_cache             | NO                             |
| long_query_time              | 15.000000                      |
| query_alloc_block_size       | 8192                           |
| query_prealloc_size          | 8192                           |
| slow_query_log               | ON                             |
| slow_query_log_file          | /data/mysqlLog/logs/mysql.slow |
+------------------------------+--------------------------------+
8 rows in set (0.00 sec)

localhost 14:51:03 [mes_prd]> show variables like '%log_timestamps%';
+----------------+--------+
| Variable_name  | Value  |
+----------------+--------+
| log_timestamps | UTC	  |
+----------------+--------+

localhost 14:49:11 [mes_prd]> set global log_timestamps = SYSTEM;
Query OK, 0 rows affected (0.00 sec)
```

## 慢sql日志分析

```
update
	SFC_SCHEDULE_EQUIPMENT tout1,
	(
	select
		tt1.ID,
		tt2.RUN_STATUS
	from
		SFC_SCHEDULE_EQUIPMENT tt1
	join (
		select
			ORG_ID ,
			EQUIPMENT_CODE,
			RUN_STATUS
		from
			(
			select
				ORG_ID ,
				EQUIPMENT_CODE,
				RUN_STATUS,
				rank() over(partition by ORG_ID,
				EQUIPMENT_CODE
			order by
				DATETIME_STATUS desc,
				ID desc) rn
			from
				EAM_EQUIPMENT_STATUS tt
			where
				1 = 1
                                                       ) t1
		where
			t1.rn = 1)tt2 
                                     on
		tt2.EQUIPMENT_CODE = tt1.EQUIPMENT_CODE
		and tt2.ORG_ID = tt1.ORG_ID
	where
		1 = 1
		and tt1.STATE = 'A'
		and tt1.SCHEDULE_ID in('353E8DE92EE342A08389964AD1884015', 'CC85D7AD316D4ABCB6062ADEE8DA5EFB', '3137C099D25D41249E8F99C9EA208E82', 'E75DB37FB39C48C4822BDE43D0A5F658', '288E56E16B2D4AE29BFB8A8E1CC06CDC')
                                    )tout2
                                    set
	tout1.DATETIME_MODIFIED = now(),
	tout1.USER_MODIFIED = 'SYS',
	tout1.IS_WORK = if(tout2.RUN_STATUS = '运行',
	'Y',
	'N'),
	tout1.IS_WORKED = if(tout2.RUN_STATUS = '运行',
	'Y',
	tout1.IS_WORKED)
where
	tout1.id = tout2.id
```

执行计划生成树：

```
-> Nested loop inner join  (cost=19115.77 rows=53723)
    -> Nested loop inner join  (cost=312.70 rows=123)
        -> Filter: ((tt1.STATE = 'A') and (tt1.SCHEDULE_ID in ('353E8DE92EE342A08389964AD1884015','CC85D7AD316D4ABCB6062ADEE8DA5EFB','3137C099D25D41249E8F99C9EA208E82','E75DB37FB39C48C4822BDE43D0A5F658','288E56E16B2D4AE29BFB8A8E1CC06CDC')) and (tt1.EQUIPMENT_CODE is not null))  (cost=269.70 rows=123)
            -> Table scan on tt1  (cost=269.70 rows=2457)
        -> Single-row index lookup on tout1 using PRIMARY (ID=tt1.ID)  (cost=0.25 rows=1)
    -> Filter: ((t1.EQUIPMENT_CODE = tt1.EQUIPMENT_CODE) and (t1.ORG_ID = tt1.ORG_ID))  (cost=0.25..109.68 rows=437)
        -> Index lookup on t1 using <auto_key0> (ORG_ID=tt1.ORG_ID, EQUIPMENT_CODE=tt1.EQUIPMENT_CODE, rn=1)
            -> Materialize  (cost=0.00..0.00 rows=0)
                -> Window aggregate: rank() OVER (PARTITION BY tt.ORG_ID,tt.EQUIPMENT_CODE ORDER BY tt.DATETIME_STATUS desc,tt.ID desc ) 
                    -> Sort: tt.ORG_ID, tt.EQUIPMENT_CODE, tt.DATETIME_STATUS DESC, tt.ID DESC  (cost=111119.85 rows=1074461)
                        -> Index scan on tt using eam_equipment_status_EQUIPMENT_CODE_IDX_U
```

关键SQL

```
select
			ORG_ID ,
			EQUIPMENT_CODE,
			RUN_STATUS
		from
			(
			select
				ORG_ID ,
				EQUIPMENT_CODE,
				RUN_STATUS,
				rank() over(partition by ORG_ID,
				EQUIPMENT_CODE
			order by
				DATETIME_STATUS desc,
				ID desc) rn
			from
				EAM_EQUIPMENT_STATUS tt
			where
				1 = 1
                                                       ) t1
		where
			t1.rn = 1)tt2 
```

“t1.rn = 1)tt2”这里查询几百万行数据之位查出排名=1 的 数据，
