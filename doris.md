
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