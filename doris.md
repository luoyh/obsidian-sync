
**doris-all-in-one**

```bash
 docker run -d --name doris -p 9030:9030 -p 8030:8030 -p 8040:8040 -p 9060:9060 apache/doris:doris-all-in-one-2.1.0
```

**gaps and island**

```sql
CREATE TABLE `iot_device_evt` (
  `device_id` bigint NOT NULL,
  `evt_time` datetime NOT NULL,
  `evt_state` int NOT NULL COMMENT '1-在线,0-离线'
) ENGINE=OLAP
UNIQUE KEY(`device_id`, `evt_time`)
COMMENT 'iot_device_evt'
DISTRIBUTED BY HASH(`device_id`, `evt_time`) BUCKETS AUTO
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);

mysql> select * from iot_device_evt order by device_id,evt_time;
+-----------+---------------------+-----------+
| device_id | evt_time            | evt_state |
+-----------+---------------------+-----------+
|         1 | 2024-12-23 00:00:00 |         0 |
|         1 | 2024-12-23 00:00:10 |         1 |
|         1 | 2024-12-23 00:00:15 |         0 |
|         1 | 2024-12-23 00:00:30 |         0 |
|         1 | 2024-12-23 00:00:35 |         1 |
|         1 | 2024-12-23 00:00:50 |         1 |
|         1 | 2024-12-23 00:01:00 |         0 |
|         2 | 2024-12-23 00:00:00 |         0 |
|         2 | 2024-12-23 00:00:10 |         0 |
|         2 | 2024-12-23 00:00:20 |         1 |
|         2 | 2024-12-23 00:00:35 |         1 |
|         2 | 2024-12-23 00:00:50 |         0 |
|         2 | 2024-12-23 00:01:00 |         1 |
|         2 | 2024-12-23 00:01:20 |         0 |
|         2 | 2024-12-23 00:01:55 |         0 |
+-----------+---------------------+-----------+
15 rows in set (0.07 sec)

-- 需要统计每个设备的在线离线时长记录
-- 如
+-----------+---------------------+-----------+-----+---------------------+------+
| device_id | st                  | evt_state | grp | et                  | dur  |
+-----------+---------------------+-----------+-----+---------------------+------+
|         1 | 2024-12-23 00:00:00 |         0 |   0 | 2024-12-23 00:00:10 |   10 |
|         1 | 2024-12-23 00:00:10 |         1 |   1 | 2024-12-23 00:00:15 |    5 |
|         1 | 2024-12-23 00:00:15 |         0 |   1 | 2024-12-23 00:00:35 |   20 |
|         1 | 2024-12-23 00:00:35 |         1 |   3 | 2024-12-23 00:01:00 |   25 |
|         1 | 2024-12-23 00:01:00 |         0 |   3 | NULL                | NULL |
|         2 | 2024-12-23 00:00:00 |         0 |   0 | 2024-12-23 00:00:20 |   20 |
|         2 | 2024-12-23 00:00:20 |         1 |   2 | 2024-12-23 00:00:50 |   30 |
|         2 | 2024-12-23 00:00:50 |         0 |   2 | 2024-12-23 00:01:00 |   10 |
|         2 | 2024-12-23 00:01:00 |         1 |   3 | 2024-12-23 00:01:20 |   20 |
|         2 | 2024-12-23 00:01:20 |         0 |   3 | NULL                | NULL |
+-----------+---------------------+-----------+-----+---------------------+------+
10 rows in set (0.09 sec)


with group_grp as (
  select 
    device_id,
    min(evt_time) st,
    evt_state,
    ROW_NUMBER() over(partition by device_id order by evt_time) -
    ROW_NUMBER() over(partition by device_id,evt_state order by evt_time) grp
  from iot_device_evt
  -- where device_id=1
  group by device_id,evt_state,grp
  order by st
)

select 
  device_id,
  st,
  evt_state,
  grp,
  lead(st,1,null) over(partition by device_id order by st) et,
  TIMESTAMPDIFF(second,st,lead(st,1,null) over(partition by device_id order by st)) dur
from group_grp
order by device_id,st;

```


**计算里程**

```sql
mysql> desc tmp_gps;
+-------------+----------+------+-------+---------+-------+
| Field       | Type     | Null | Key   | Default | Extra |
+-------------+----------+------+-------+---------+-------+
| vehicle_id  | bigint   | Yes  | true  | NULL    |       |
| device_time | datetime | Yes  | true  | NULL    |       |
| speed       | int      | Yes  | false | NULL    | NONE  |
| mileage     | bigint   | Yes  | false | NULL    | NONE  |
| longitude   | bigint   | Yes  | false | NULL    | NONE  |
| latitude    | bigint   | Yes  | false | NULL    | NONE  |
+-------------+----------+------+-------+---------+-------+
6 rows in set (0.04 sec)

mysql> select * from tmp_gps;
+------------+---------------------+-------+---------+-----------+----------+
| vehicle_id | device_time         | speed | mileage | longitude | latitude |
+------------+---------------------+-------+---------+-----------+----------+
|          1 | 2025-01-01 00:00:10 |     1 |       3 | 106194254 | 30003730 |
|          1 | 2025-01-01 00:00:20 |     1 |       4 | 106223634 | 30029164 |
|          1 | 2025-01-01 00:00:30 |     1 |       3 | 106245674 | 30048239 |
|          1 | 2025-01-01 00:00:40 |     1 |       4 | 106275068 | 30073673 |
|          1 | 2025-01-01 00:00:50 |     1 |       3 | 106297117 | 30092748 |
|          1 | 2025-01-01 00:01:00 |     1 |       5 | 106333878 | 30124539 |
|          1 | 2025-01-01 00:01:10 |     1 |       6 | 106378008 | 30162687 |
|          1 | 2025-01-01 00:01:20 |     1 |       1 | 106385363 | 30169046 |
|          1 | 2025-01-01 00:01:30 |     1 |       1 | 106392719 | 30175404 |
+------------+---------------------+-------+---------+-----------+----------+

mysql> SELECT
    a.vehicle_id,
    a.device_time,
    a.longitude AS longitude,
    a.latitude AS latitude,
    SUM(
      CASE
          WHEN ROW_NUMBER() OVER (PARTITION BY a.vehicle_id ORDER BY a.device_time) = 1 THEN 0.0  -- 第一个点，累计里程为 0
          ELSE point_mileage
      END
    ) OVER (PARTITION BY a.vehicle_id ORDER BY a.device_time) / 1000
      as cumulative_mileage1,
    SUM(point_mileage) OVER (PARTITION BY a.vehicle_id ORDER BY a.device_time) / 1000
      as cumulative_mileage
FROM (
    SELECT
        a.vehicle_id,
        a.device_time,
        a.longitude / 1000000 AS longitude,
        a.latitude / 1000000 AS latitude,
        ST_Distance_Sphere( -- 计算距离
          nvl((lag(a.longitude/1000000,1,null) over (partition by a.vehicle_id order by a.device_time)),
            a.longitude/1000000), -- 获取前一个经纬度,如果为空则用当前经纬度
          nvl((lag(a.latitude/1000000,1,null) over (partition by a.vehicle_id order by a.device_time)),
            a.latitude/1000000),
          a.longitude/1000000,
          a.latitude/1000000) as point_mileage
    FROM tmp_gps a
    order by vehicle_id,device_time
) AS a
ORDER BY vehicle_id,device_time;
+------------+---------------------+------------+-----------+---------------------+--------------------+
| vehicle_id | device_time         | longitude  | latitude  | cumulative_mileage1 | cumulative_mileage |
+------------+---------------------+------------+-----------+---------------------+--------------------+
|          1 | 2025-01-01 00:00:10 | 106.194254 |  30.00373 |                   0 |                  0 |
|          1 | 2025-01-01 00:00:20 | 106.223634 | 30.029164 |    4.00002953842755 |   4.00002953842755 |
|          1 | 2025-01-01 00:00:30 | 106.245674 | 30.048239 |  7.0000159752416735 | 7.0000159752416735 |
|          1 | 2025-01-01 00:00:40 | 106.275068 | 30.073673 |  11.000099925105134 | 11.000099925105134 |
|          1 | 2025-01-01 00:00:50 | 106.297117 | 30.092748 |  14.000024339033235 | 14.000024339033235 |
|          1 | 2025-01-01 00:01:00 | 106.333878 | 30.124539 |  19.000064347873266 | 19.000064347873266 |
|          1 | 2025-01-01 00:01:10 | 106.378008 | 30.162687 |  25.000097799657283 | 25.000097799657283 |
|          1 | 2025-01-01 00:01:20 | 106.385363 | 30.169046 |   26.00006917546629 |  26.00006917546629 |
|          1 | 2025-01-01 00:01:30 | 106.392719 | 30.175404 |   26.99999765205352 |  26.99999765205352 |
|          1 | 2025-01-01 00:01:40 | 106.414792 |  30.19448 |  30.000086549124042 | 30.000086549124042 |
|          1 | 2025-01-01 00:01:50 | 106.444229 | 30.219913 |    34.0000523172636 |   34.0000523172636 |
+------------+---------------------+------------+-----------+---------------------+--------------------+


-- 从第一行到最后一行累加
select sum(duration) over(
  partition by vehicle_id 
  order by start_time 
  rows between UNBOUNDED PRECEDING and UNBOUNDED following) duration,
  last_value(acc_status) over(partition by vehicle_id order by start_time)
   from end_time where acc_status=1;
   
-- 从第一行到当前行
select sum(duration) over(
  partition by vehicle_id 
  order by start_time 
  rows between 1 and current row) duration,
  last_value(acc_status) over(partition by vehicle_id order by start_time)
   from end_time where acc_status=1;

-- 计算acc开时长
with state_groups as (
  select
    vehicle_id,
    device_time,
    bitand(status_bit, 1) AS acc_status,
    ROW_NUMBER() OVER(PARTITION BY vehicle_id ORDER BY device_time) - 
    ROW_NUMBER() OVER(PARTITION BY vehicle_id, bitand(status_bit, 1) ORDER BY device_time) AS grp
  from rm_vehicle_gps_info 
  WHERE device_time BETWEEN '2024-12-24 11:55:02' AND '2024-12-24 11:56:00'
  and message_id=512
  -- and bitand(status_bit,1)=0
  and vehicle_id=1783376234900697089
  order by device_time
)
, group_grp as (

  select 
    vehicle_id,
    min(device_time) start_time,
    acc_status
  from state_groups
  group by vehicle_id,grp,acc_status
  order by start_time
)
, end_time as (
  select 
    vehicle_id,
    start_time,
    lead(start_time, 1, null) over(partition by vehicle_id order by start_time) end_time,
    TIMESTAMPDIFF(
      SECOND,
      start_time,
      ifnull(
        lead(start_time, 1, null) over(partition by vehicle_id order by start_time),
        '2024-12-24 11:56:00')
      ) as duration,
    acc_status
  from group_grp
  order by start_time
)
select 
  sum(duration) over(partition by vehicle_id order by start_time rows between UNBOUNDED PRECEDING and UNBOUNDED following) duration,
  last_value(acc_status) over(partition by vehicle_id order by start_time)
   from end_time where acc_status=1
;
```


```sql
-- 轨迹完整率
-- 速度小于2km计有效里程, 完整里程
-- 速度连续>=2km, <=10km, 前5段计入有效里程; 计入完整里程;
-- 速度>10km, 只有一个点, 计入完整里程
-- 速度连续>10km, 只记录最后一个点到前面有效轨迹点的距离,计入完整里程
-- 比如, a, b, c, d, e, f 6个点,
-- a->b = 3, b -> c = 12, c ->d = 14, d -> e = 11, e -> f =8,
-- 完整里程是a->b + b -> e + e ->f的距离.其中c,d是的点是漂移点

            with prev_data as (
                select 
                  a.company_id,
                  a.id vehicle_id,
                  a.plate,
                  ifnull(b.end_time,'2025-01-08 00:00:00') device_time,
                  b.end_lon as longitude,
                  b.end_lat as latitude,
                  null mileage
                from 
                ods_om_vehicle a 
                left join ob_vehicle_summary_day b on a.id=b.vehicle_id 
                and b.indate='2025-01-08'
                where a.del_flag='0'
                -- and a.id=1876522409877569537
            )
            , gps_day as (
                select
                  b.company_id,
                  a.vehicle_id,
                  b.plate,
                  a.device_time,
                  a.longitude,
                  a.latitude,
                  -- 终端里程
                  cast(get_json_string(a.attributes,'$.1') as bigint) mileage
                from rm_vehicle_gps_info a
                join ods_om_vehicle b on a.vehicle_id=b.id
                WHERE a.device_time BETWEEN '2025-01-09 00:00:00' AND '2025-01-09 23:59:59'
                and a.message_id=512
                -- and a.vehicle_id=1876522409877569537
              union all
                select 
                  *
                from prev_data
            )
            , prev_gps as (
              select 
                company_id,
                vehicle_id,
                plate,
                device_time,
                longitude,
                latitude,
                -- 前一个经纬度
                lag(longitude,1,null) over(partition by vehicle_id order by device_time) prev_lon,
                lag(latitude,1,null) over(partition by vehicle_id order by device_time) prev_lat,
                mileage,
                -- 前一个终端里程
                lag(mileage,1,null) over(partition by vehicle_id order by device_time) prev_mileage
              from gps_day 
              where 
                longitude is not null 
                and latitude is not null 
                and longitude > 0 and longitude <= 180000000
                and latitude > 0 and latitude <= 90000000
            )
            , calc_distance as (
              select 
                company_id,
                vehicle_id,
                plate,
                device_time,
                longitude,
                latitude,
                prev_lon,
                prev_lat,
                -- 根据经纬度计算距离
                ST_Distance_Sphere(
                  longitude/1000000,
                  latitude/1000000,
                  nvl(prev_lon,longitude)/1000000,
                  nvl(prev_lat,latitude)/1000000
                ) distance,
                mileage,
                -- 两个点的终端里程距离
                greatest((mileage-nvl(prev_mileage,mileage))*100,0) prev_mileage 
              from prev_gps
            )
            , seg_mileage as (
              select 
                company_id,
                vehicle_id,
                plate,
                device_time,
                longitude,
                latitude,
                prev_lon,
                prev_lat,
                distance,
                row_number() over(partition by vehicle_id order by device_time) rn,
                -- 根据距离分段
                case 
                  -- 小于2km是连续里程
                  when distance < 2000 then 0
                  -- [2,10]是连续里程
                  when distance between 2000 and 10000 then 1
                  -- >10 是漂移里程
                  else 2
                end seg1,
                -- 根据分段设置行号
                row_number() over(
                  partition by vehicle_id,
                  case 
                    when distance < 2000 then 0
                    when distance between 2000 and 10000 then 1
                    else 2
                  end
                  order by device_time
                ) rn1,
                
                -- 下面和上面的distance一样, 只是是终端里程
                prev_mileage mileage,
                lag(prev_mileage,1,0) over(partition by vehicle_id order by device_time) prev2,
                case 
                  when prev_mileage < 2000 then 0
                  when prev_mileage between 2000 and 10000 then 1
                  else 2
                end seg2,
                row_number() over (
                  PARTITION by vehicle_id,
                  case 
                    when prev_mileage < 2000 then 0
                    when prev_mileage between 2000 and 10000 then 1
                    else 2
                  end
                  order by device_time
                ) as rn2    
              from 
              calc_distance
            )
            , count_idx as (
              select 
                *,
                -- gps 连续段
                rn-rn1 island1,
                -- 终端里程连续段
                rn-rn2 island2,
                -- gps 连续段行号
                count() over(partition by vehicle_id,rn-rn1,seg1 order by device_time) idx1,
                -- gps 连续段数量
                count() over(partition by vehicle_id,rn-rn1,seg1) total1,
                -- 终端连续段行号
                count() over(partition by vehicle_id,rn-rn2,seg2 order by device_time) idx2,
                -- 终端连续段数量
                count() over(partition by vehicle_id,rn-rn1,seg1) total2
                from seg_mileage
            )
            , calc_final as (
               select 
                  *,
                  -- gps有效里程
                  case 
                    -- 小于2km的是有效
                    when seg1 = 0 then distance
                    -- [2,10]只取前5段
                    when seg1 = 1 and idx1 <= 5 then distance
                    -- [2,10]只取前5端
                    when seg1 = 1 and idx1 > 5 then 0
                    -- 大于10的无效
                    when seg1 = 2 then 0
                    else 0
                  end valid1,
                  -- gps完整里程
                  case 
                    -- 小于2km和[2,10]的 都是完整里程
                    when seg1 = 0 or seg1 = 1 then distance
                    -- 大于10公里,且只有1个点的是完整里程
                    when seg1 = 2 and total1 = 1 then distance 
                    -- 连续多个点大于10km,取最后一个点的经纬度到前面小于10km的点的距离
                    when seg1 = 2 and total1 > 1 and idx1 = 1 then 
                      ST_Distance_Sphere(
                        last_value(longitude) over (
                          partition by vehicle_id,seg1,island1
                          order by device_time
                          rows BETWEEN unbounded preceding and unbounded following
                        )/1000000,
                        last_value(latitude) over (
                          partition by vehicle_id,seg1,island1
                          order by device_time
                          rows BETWEEN unbounded preceding and unbounded following
                        )/1000000,
                        first_value(prev_lon) over (
                          partition by vehicle_id,seg1,island1
                          order by device_time
                          rows BETWEEN unbounded preceding and unbounded following
                        )/1000000,
                        first_value(prev_lat) over (
                          partition by vehicle_id,seg1,island1
                          order by device_time
                          rows BETWEEN unbounded preceding and unbounded following
                        )/1000000
                      )
                    -- 连续多个点10km,只算第一个点,后面的设置0
                    when seg1 = 2 and total1 > 1 and idx1 > 1 then 0
                    else 0
                  end all1,
                  
                  -- 下面是终端的算法和gps一样
                  case 
                    when seg2 = 0 then mileage
                    when seg2 = 1 and idx2 <= 5 then mileage
                    when seg2 = 1 and idx2 > 5 then 0
                    when seg2 = 2 then 0
                    else 0
                  end valid2,
                  
                  case 
                    when seg2 = 0 or seg2 = 1 then mileage
                    when seg2 = 2 and total2 = 1 then mileage 
                    when seg1 = 2 and total2 > 1 and idx2 = 1 then 
                      last_value(mileage) over (
                          partition by vehicle_id,seg2,island2
                          order by device_time
                          rows BETWEEN unbounded preceding and unbounded following
                        )
                        -
                      first_value(prev2) over (
                        partition by vehicle_id,seg2,island2
                        order by device_time
                        rows BETWEEN unbounded preceding and unbounded following
                      )
                    when seg2 = 2 and total2 > 1 and idx2 > 1 then 0
                    else 0
                  end all2
                  
                from count_idx
            )
            select
              vehicle_id,
              -- gps有效里程
              sum(valid1) valid1,
              -- gps完整里程
              sum(all1) all1,
              -- 终端有效里程
              sum(valid2) valid2,
              -- 终端完整里程
              sum(all2) all2
            from calc_final
            group by vehicle_id
            order by vehicle_id
            
```