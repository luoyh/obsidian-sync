

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


## flink run on docker

```yaml

# Dockerfile
# Use the official Flink image as base, adjust the tag as needed (e.g., 1.19-scala_2.12)
FROM flink:1.16.3

# Download the Flink Kafka SQL Connector JAR and place it in the Flink lib directory
# This example uses a specific version; check the Flink documentation for the correct one
COPY lib/* /opt/flink/lib/


# docker-compose.yml
version: "2.2"
services:
  jobmanager:
#image: flink:1.16.3
    build: .
    ports:
      - "17013:8081"
    command: jobmanager
    extra_hosts:
      - "tuqiaoxing-kafka.tqxing-test:192.168.1.11"
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager

  taskmanager:
#   image: flink:1.16.3
    build: .
    depends_on:
      - jobmanager
    command: taskmanager
    extra_hosts:
      - "tuqiaoxing-kafka.tqxing-test:192.168.1.11"
    scale: 2
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2                                                                                                     
```

### start

```bash
# lib
[root@localhost flink]# ll lib/
total 66792
-rw-r--r--. 1 root root   248892 Dec  2 10:42 flink-connector-jdbc-1.16.3.jar
-rw-r--r--. 1 root root 40950669 Jan 22 14:44 flink-doris-connector-1.16-25.1.0.jar
-rw-r--r--. 1 root root  5553899 Oct 23  2024 flink-sql-connector-kafka-1.16.2.jar
-rw-r--r--. 1 root root 21637117 Jan 22 14:54 flink-sql-connector-mysql-cdc-3.5.0.jar


# start
docker-compose -f flink-compose.yml up -d
```