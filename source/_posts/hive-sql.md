---
title: hive sql 调优
date: 2021-03-06 14:24:00
tags: [big-data]
---


hive 其实是用 sql 来开发的 mapreduce 任务，其中的一些有优化技巧和 sql 优化也有相同的地方

在经过一周的 sql boy 历练以后，写 sql 的水平也算是更上一层楼了



虽然说hive 跑的都是t-1的离线数据，不怎么需要在意运行时间，但是比较 yarn 集群资源有限，使用尽量少的资源来完成尽量多的工作也是很棒的！这里总结一下一些 hql 优化上的技巧

<!-- more -->

**1. 所有分区表都需要指定分区查询**

```sql
-- 反例
select id, user_id from partition_table_name; -- 执行起来会很慢，因为需要扫描所有分区
-- 正例
select id,user_id from partition_table_name where date = '${date}' -- 只扫描制定的 date 分区
```



**2. 不要偷懒使用 * 查全部数据字段，需要什么字段拿什么**

对于一些大宽表来说，可能有七八十个字段，但是实际上只用到了其中两三个字段，造成了严重的资源浪费

```sql
-- 反例
select
	su.star_id,
	au.user_id,
	au.device_id
from 
	star_user_profile as su
join
	(
  	select
    	*
    from
    	aweme_user_profile  -- aweme_user_profile 可能有上百个字段
    where date = ${date}
    and user_id > 0
  ) as au
 on au.user_id = su.core_user_id;
 
 -- 正例
 select
	su.star_id,
	au.user_id,
	au.device_id
from 
	star_user_profile as su
join
	(
  	select
    	user_id,
    	device_id
    from
    	aweme_user_profile 
    where date = ${date}
    and user_id > 0
  ) as au
 on au.user_id = su.core_user_id;
```



**3. 分区条件都放到表后，join 前**

```sql
-- 反例
select 
	ui.id, 
	ui.user_id,
	ui.device_id,
	di.device_brand
from 
	partition_user_profile_info as ui 
left join 
	partition_device_info as di
on 
	di.device_id = ui.device_id and di.device_id > 0
where ui.date = '${date}' and di.date = '${date}'

-- 正例
select
	ui.id,
	ui.user_id,
	ui.device_id,
	di.device_brand
from 
	(select id, user_id, device_id from partition_user_profile_info where date = '${date}') as ui
left join
	(select device_id, device_brand from partition_device_info where date = '${date}') as di
on di.device_id = ui.device_id
```



**4. join时，改表的条件紧跟当前表，不要join一堆表后都放到一个on里面**

```sql
-- 反例
select
    g.star_id,
    g.gender_distribution,
    a.age_distribution,
    p.province_distribution,
    c.city_distribution,
    d.device_brand_distribution,
    unix_timestamp() as update_time
from
    p_gender_dist as g
join
    p_age_dist as a 
join
    p_province_dist as p 
join 
    p_city_dist as c
join
    p_device_brand_dist as d 
on 
	g.core_user_id = a.core_user_id
	and a.core_user_id = p.core_user_id
	and p.core_user_id = c.core_user_id
	and c.core_user_id = d.core_user_id
    
-- 正例
select
    g.star_id,
    g.gender_distribution,
    a.age_distribution,
    p.province_distribution,
    c.city_distribution,
    d.device_brand_distribution,
    unix_timestamp() as update_time
from
    p_gender_dist as g
join
    p_age_dist as a on g.core_user_id = a.core_user_id
join
    p_province_dist as p on a.core_user_id = p.core_user_id
join 
    p_city_dist as c on p.core_user_id = c.core_user_id
join
    p_device_brand_dist as d on c.core_user_id = d.core_user_id
```



**5. hive 参数调优**

[啥都比不过官方文档系列~](https://spark.apache.org/docs/2.3.4/configuration.html)



- 在碰到一些临时表特别大的场景下，可能会产生某个 任务 内存不够的场景，添加内存参数；对于一些任务oom之后，会不断重试失败，浪费很多时间


```sql
set spark.driver.memory = 80g;
set spark.executor.memory = 80g;
set spark.driver.maxResultSize=16g;  -- default 1g
```

![image-20210308202508074](1.png)

- yarn的资源分配 https://zhuanlan.zhihu.com/p/335881182  yarn -> nodemanager -> container

```sql
set spark.vcore.boost.ratio = 1;
```



- 利用好重试机制

```sql
set spark.shuffle.io.maxRetries=1; 
set spark.shuffle.io.retryWait=0s; 
set spark.network.timeout=120s;
```



- 使用 external shuffle service 提高性能

```sql
set spark.shuffle.hdfs.enabled
```





**6. sql 的严谨性**

- 一条记录里经常会有各种空字段，记得对空字段进行处理

```sql
-- 反例
concat(":", tmp.province, cast(tmp.cnt as string))

-- 正例
concat_ws(":", case when tmp.province is NULL or tmp.province="" then "unknown" else tmp.province end, cast(tmp.cnt as string))
```



- 做数据统计的时候，不要过分信任数据的准确性，需要自己做一层判断，例如

```sql
-- 反例
(select author_id, item_id, user_id from aweme_user_item_info) as item_reach;

-- 正例
(select author_id, item_id, user_id from aweme_user_item_info where item_id > 0) as item_reach -- 排除 item_id = 0 的脏数据
```



---

**其他常用函数**

- 获取group by 后每一组对该列的排位。使用场景举例：获取所有博主的最近 2 条微博

table: author_item_dict

| author_id | weibo_id | create_time |
| --------- | -------- | ----------- |
| 1         | 23489032 | 1615217421  |
| 1         | 12309780 | 1615217419  |
| 2         | 43242511 | 1615217418  |
| 1         | 45435346 | 1615217422  |



```sql
select
	author_id,
	weibo_id
from 
	(
  	select
    	author_id,
    	weibo_id,
    	row_number over(partition by author_id order by create_time) as rn
   	from
    	author_item_dict
  ) as ai
  where 
  	ai.rn >= 2
```



result:

| author_id | weibo_id |
| --------- | -------- |
| 1         | 12309780 |
| 1         | 23489032 |
| 2         | 43242511 |

