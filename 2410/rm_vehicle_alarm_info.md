
# 定义

```sql
CREATE TABLE `rm_vehicle_alarm_info` (
  `id` varchar(32) NOT NULL COMMENT '报警ID，唯一编号',
  `vehicle_id` bigint NOT NULL COMMENT '车牌ID',
  `client_id` varchar(40) NOT NULL COMMENT '设备编号',
  `plate` text NULL COMMENT '车牌号',
  `arm_time_start` datetime NULL COMMENT '报警开始时间',
  `arm_time_end` datetime NULL COMMENT '报警结束时间',
  `arm_type` text NULL COMMENT '报警类型',
  `arm_info` text NULL COMMENT '报警信息',
  `arm_desc` text NULL COMMENT '报警描述',
  `status_start` int NULL COMMENT '报警开始状态信息',
  `status_end` int NULL COMMENT '报警结束状态信息',
  `image_info` text NULL DEFAULT "" COMMENT '图片信息',
  `handle_status` tinyint NULL DEFAULT "0" COMMENT '处理状态 0未处理,1已处理',
  `handle_info` text NULL DEFAULT "" COMMENT '处理信息',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP,
  `handle_user_id` bigint NULL DEFAULT "0" COMMENT '处理用户ID',
  `start_position` text NULL DEFAULT "" COMMENT '开始地理位置',
  `end_position` text NULL DEFAULT "" COMMENT '结束地理位置',
  `handle_way` text NULL COMMENT '处理方式',
  `handle_up` text NULL COMMENT '处理上报',
  `arm_detail` text NULL DEFAULT "" COMMENT '附加消息',
  `manual_handle_type` text NULL DEFAULT "" COMMENT '人工确认报警类型',
  `handle_model` text NULL COMMENT '处理模块',
  `msg_type` text NULL COMMENT '消息类型'
) ENGINE=OLAP
UNIQUE KEY(`id`)
COMMENT '报警信息记录表，id由vehicle_id，arm_time_start，arm_type 换算为hash值'
DISTRIBUTED BY HASH(`id`) BUCKETS AUTO
PROPERTIES (
"replication_allocation" = "tag.location.default: 1",
"min_load_replica_num" = "-1",
"is_being_synced" = "false",
"storage_medium" = "hdd",
"storage_format" = "V2",
"inverted_index_storage_format" = "V1",
"estimate_partition_size" = "2G",
"enable_unique_key_merge_on_write" = "true",
"light_schema_change" = "true",
"disable_auto_compaction" = "false",
"enable_single_replica_compaction" = "false",
"group_commit_interval_ms" = "10000",
"group_commit_data_bytes" = "134217728",
"enable_mow_light_delete" = "false"
);;
```

> 
> `id`算法`id = md5(vehicle_id, arm_time_start, arm_type)`
> 所以可以去掉id, 使用这3个做`unique key`.
> 后面需要加上`arm_source (报警来源,int, 设备/平台/频繁)`.

测试的arm表数据量: `59818982`

# 目前的SQL
## listAlarmInfo

**SQL**
```sql
    SELECT t.id,
		t.vehicle_id,
		t.client_id,
		t.plate,
		t.arm_time_start,
		t.arm_time_end,
		t.arm_type,
		t.arm_info,
		t.arm_desc,
		t.status_start,
		t.status_end,
		t.image_info,
		t.handle_status,
		t.handle_info,
		t.create_time,
		t.handle_user_id,
		t.start_position,
		t.end_position,
		t.msg_type,
		t.handle_way,
		t.handle_up,
		t.handle_model
		FROM rm_vehicle_alarm_info t
		join ods_om_vehicle_user u on u.vehicle_id = t.vehicle_id
		WHERE u.user_id = #{userId} and u.del_flag = 0
		<if test="query.handleStatus != null">
			and t.handle_status = #{query.handleStatus}
		</if>
		<if test="query.getVehicleIds() != null and query.getVehicleIds().size >0">
			<foreach collection="query.getVehicleIds()" item="item" open="and t.vehicle_id in (" separator="," close=")">
				#{item}
			</foreach>
		</if>
		<if test="query.getAlarmType() != null and query.getAlarmType().size >0">
			<foreach collection="query.getAlarmType()" item="item" open="and t.arm_type in (" separator="," close=")">
				#{item}
			</foreach>
		</if>
		<if test="query.startTime != null">
			and t.create_time >= #{query.startTime}
		</if>
		<if test="query.endTime != null">
			and t.create_time &lt;= #{query.endTime}
		</if>
		order by t.arm_time_start desc
```

> 此SQL是一个分页查询的SQL, 使用到了`arm_time_start`排序,
> `ods_om_vehicle_user`数据较小, 可以忽略. 
> 在开发环境测试了1点几秒就能返回结果.
> `count(*)` 只需要300ms



## getAlarmPageList

```sql
SELECT t.id,
		t.vehicle_id,
		t.client_id,
		t.plate,
		t.arm_time_start,
		t.arm_time_end,
		t.arm_type,
		t.arm_info,
		t.arm_desc,
		t.status_start,
		t.status_end,
		t.image_info,
		t.handle_status,
		t.handle_info,
		t.create_time,
		t.handle_user_id,
		t.start_position,
		t.end_position,
		t.msg_type,
		t.handle_way,
		t.handle_up,
		t.handle_model,
		t.manual_handle_type,
		d.protocol_type
		FROM rm_vehicle_alarm_info t
		join ods_om_vehicle_user u on u.vehicle_id = t.vehicle_id
		join ods_om_device d on t.client_id = d.device_no
		WHERE u.user_id = #{userId} and u.del_flag = 0
		<if test="query.startTime != null">
			and t.arm_time_start >= #{query.startTime}
		</if>
		<if test="query.vehicleId != null ">
			and t.vehicle_id = #{query.vehicleId}
		</if>
		<if test="query.vehicleIds != null and query.vehicleIds.size()>0">
            and t.vehicle_id in (
            <foreach collection="query.vehicleIds" item="e" separator=",">
                #{e}
            </foreach>
            )
		</if>
		<if test="query.clientId != null ">
			and t.client_id = #{query.clientId}
		</if>
        <if test="query.endTime != null">
			and t.arm_time_start &lt;= #{query.endTime}
		</if>

		<if test="query.handleStatus != null">
			and t.handle_status = #{query.handleStatus}
		</if>
		<if test="query.handleWay != null">
			and t.handle_way = #{query.handleWay}
		</if>
		<if test="query.handleUp != null">
			and t.handle_up = #{query.handleUp}
		</if>

		<if test="query.armType != null">
			and t.arm_type = #{query.armType}
		</if>
		order by t.arm_time_start desc
```

> 和上面一样


## selectAlarmSummaryPage

```sql
select vehicle_id vehicleId,
			   SUM(CASE WHEN arm_type='overspeedAlarm' THEN 1 ELSE 0 END) as overspeedAlarmCount,
			   SUM(CASE WHEN arm_type='overspeedAlarm' and handle_status=1  THEN 1 ELSE 0 END) as overspeedAlarmDealCount,
			   SUM(CASE WHEN arm_type='fatigueAlarm' THEN 1
						WHEN arm_type='fatigueEarlyAlarm' THEN 1 ELSE 0 END) as tiredAlarmCount,
			   SUM(CASE WHEN arm_type='fatigueAlarm' and handle_status=1 THEN 1
						WHEN arm_type='fatigueEarlyAlarm' and handle_status=1  THEN 1 ELSE 0 END) as tiredAlarmDealCount,
			   SUM(CASE WHEN arm_type='emergentAlarm' THEN 1 ELSE 0 END) as emergentAlarmCount,
			   SUM(CASE WHEN arm_type='emergentAlarm' and handle_status=1  THEN 1 ELSE 0 END) as emergentAlarmDealCount,
			   SUM(CASE WHEN arm_type='dangerousAlarm' THEN 1 ELSE 0 END) as dangerousAlarmCount,
			   SUM(CASE WHEN arm_type='dangerousAlarm' and handle_status=1  THEN 1 ELSE 0 END) as dangerousAlarmDealCount
		from rm_vehicle_alarm_info
		WHERE 1=1
		<if test="query.startTime != null">
			and arm_time_start >= #{query.startTime}
		</if>
		<if test="query.vehicleIds != null and query.vehicleIds.size()>0">
			and vehicle_id in
			<foreach collection="query.vehicleIds" item="item" open="(" separator="," close=")">
				#{item}
			</foreach>
		</if>
		<if test="query.clientId != null ">
			and client_id = ${query.clientId}
		</if>
		<if test="query.endTime != null">
		and arm_time_start &lt;= #{query.endTime}
		</if>
		<if test="query.handleStatus != null">
			and handle_status = #{query.handleStatus}
		</if>
		<if test="query.handleWay != null">
			and handle_way = #{query.handleWay}
		</if>
		GROUP BY vehicle_id
```

> 单表统计, 耗时200ms左右, 
> 分组也是按照`vehicle_id`, 即可以优化表结构后理论上更快.



