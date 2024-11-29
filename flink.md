

```bash
# 显示任务
> flink list -m localhost:8081

```

## flink-doris-connector

### 使用DataStream方式

```java

        DorisSink.Builder<RowData> builder = DorisSink.builder();
        DorisOptions.Builder dorisBuilder = DorisOptions.builder();
        dorisBuilder.setFenodes(args.getDorisHost() + ":" + args.getDorisPort())
                 .setTableIdentifier(args.getDorisDb() + "." + Args.TABLE_GPS_NAME)
                 .setUsername(args.getDorisUser())
                 .setPassword(args.getDorisPassword());

        
        Properties properties = new Properties();
        // 指定处理json类型数据
        properties.setProperty("format", "json");
        properties.setProperty("read_json_by_line", "true");
        // 设置允许只更新部分列, 否则upsert不生效, 会直接覆盖
        properties.setProperty("partial_columns", "true");
        // 更新的列
        properties.setProperty("columns", "vehicle_id,client_id,device_time,message_id,serial_no,warn_bit,status_bit,latitude,longitude,altitude,speed,status,direction,attributes,create_time");
        DorisExecutionOptions.Builder executionBuilder = DorisExecutionOptions.builder();
        executionBuilder.setLabelPrefix("flink_gps_label_" + args.getLabelId()) //streamload label prefix
                         .setDeletable(false)
                         .disable2PC()
                         .setStreamLoadProp(properties); //streamload params
        
        builder
        .setDorisReadOptions(DorisReadOptions.builder().build())
        .setDorisExecutionOptions(executionBuilder.build())
        .setDorisOptions(dorisBuilder.build())
        .setSerializer(RowDataSerializer.builder()
                .setFieldNames(new String[] {
                        "vehicle_id",
                        "client_id",
                        "device_time",
                        "message_id",
                        "serial_no",
                        "warn_bit",
                        "status_bit",
                        "latitude",
                        "longitude",
                        "altitude",
                        "speed",
                        "status",
                        "direction",
                        "attributes",
                        "create_time"
                })
                .setType("json")
                .setFieldType(new DataType[] {
                        DataTypes.BIGINT(),
                        DataTypes.STRING(),
                        DataTypes.STRING(),
                        DataTypes.INT(),
                        DataTypes.INT(),
                        DataTypes.BIGINT(),
                        DataTypes.BIGINT(),
                        DataTypes.BIGINT(),
                        DataTypes.BIGINT(),
                        DataTypes.INT(),
                        DataTypes.INT(),
                        DataTypes.INT(),
                        DataTypes.INT(),
                        DataTypes.STRING(),
                        DataTypes.STRING(),
                })
                .build());
        ds.map(new MapFunction<GPS, RowData>() {
            private static final long serialVersionUID = 1L;

            @Override
            public RowData map(GPS gps) throws Exception {
                GenericRowData ps = new GenericRowData(15);
                int index = 0;
                ps.setField(index ++, gps.getVehicle().getId());
                ps.setField(index ++, StringData.fromString(gps.getSimNo()));
                if (gps.getMessageId() == Cons.MSG_SIGN) {
                    if (null != gps.getB0702() && null != gps.getB0702().getTimeMs()) {
                        ps.setField(index ++, StringData.fromString(DateUtils.format(gps.getB0702().getTimeMs())));
                    } else {
                        ps.setField(index ++, StringData.fromString(DateUtils.format(gps.getTimeMs())));
                    }
                } else {
                    ps.setField(index ++, StringData.fromString(DateUtils.format(gps.getTimeMs())));
                }
                ps.setField(index ++, gps.getMessageId());
                ps.setField(index ++, Utils.primitive(gps.getSn()));
                ps.setField(index ++, Utils.primitive(gps.getAlarmBits()));
                ps.setField(index ++, Utils.primitive(gps.getStatusBits()));
                ps.setField(index ++, Utils.primitive(gps.getLatitude()));
                ps.setField(index ++, Utils.primitive(gps.getLongitude()));
                ps.setField(index ++, Utils.primitive(gps.getAltitude()));
                ps.setField(index ++, Utils.primitive(gps.getSpeed()));
                ps.setField(index ++, Utils.primitive(gps.status()));
                ps.setField(index ++, Utils.primitive(gps.getDirection()));
                // attributes
                ps.setField(index ++, StringData.fromString(gps.attributes()));
                ps.setField(index ++, StringData.fromString(DateUtils.format(System.currentTimeMillis())));
                return ps;
            }
            
        })
        .sinkTo(builder.build())
        .setParallelism(args.getParallelismGPS() > 0 ? args.getParallelismGPS() : -1)
        ;
    
```