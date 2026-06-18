

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


## docker-compose

```ymlversion: "3.9"
# 可选的dinky
# DINKY_VERSION=1.2.5
# FLINK_VERSION=1.20.5
services:
  dinky:
    image: "dinkydocker/dinky-standalone-server:${DINKY_VERSION}-flink${FLINK_VERSION}"
    container_name: dinky
    ports:
      - "18888:8888"
    env_file: .env
    networks:
      - dinky
    volumes:
      - ${CUSTOM_JAR_PATH}:/opt/dinky/customJar/

  jobmanager:
    image: flink:1.20-scala_2.12
    ports:
      - "18081:8081"
    command: jobmanager
    env_file: .env
    networks:
      - dinky
    volumes:
      - ${CUSTOM_JAR_PATH}:/opt/flink/lib/customJar/
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager        

  taskmanager:
    image: flink:1.20-scala_2.12
    depends_on:
      - jobmanager
    command: taskmanager
    env_file: .env
    scale: 1
    networks:
      - dinky
    volumes:
      - ${CUSTOM_JAR_PATH}:/opt/flink/lib/customJar/
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2      

  sql-client:
    image: flink:1.20-scala_2.12
    command: bin/sql-client.sh
    networks:
      - dinky
    env_file: .env
    volumes:
      - ${CUSTOM_JAR_PATH}:/opt/flink/lib/customJar/
    depends_on:
      - jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        rest.address: jobmanager

networks:
  dinky:

```

.env
```bash
root@ubuntu:/data/local/dinky# cat .env 

DINKY_VERSION=1.2.5
FLINK_VERSION=1.20

# 自定义其他依赖路径，会挂载到dinky和flink容器中
CUSTOM_JAR_PATH=/data/local/dinky/libs
FLINK_PROPERTIES="jobmanager.rpc.address: jobmanager"
TZ=Asia/Shanghai

# mysql,h2,pgsql
# DB_ACTIVE=h2
DB_ACTIVE=mysql

# use h2 db localtion file path
#H2_DB=./tmp/db/h2

## Mysql Config
## 如果 DB_ACTIVE 配置为mysaql，请修改下面配置，否则忽略
MYSQL_ADDR=172.17.0.1:13306
MYSQL_DATABASE=dinky
MYSQL_USERNAME=dinky
MYSQL_PASSWORD=dinky

```

使用flink-cdc同步整库从mysql到doris

```bash
root@ubuntu:/data/local/dinky# ll libs/
flink-cdc-common-3.5.0.jar
flink-cdc-dist-3.5.0.jar
flink-cdc-pipeline-connector-doris-3.5.0.jar
flink-cdc-pipeline-connector-mysql-3.5.0.jar
flink-sql-connector-mysql-cdc-3.5.0.jar
mysql-connector-j-9.6.0.jar



```

flink-cdc mysql-redis 提交任务提示:
```
Caused by: java.lang.ClassCastException: cannot assign instance of org.apache.doris.flink.cfg.DorisOptions to field org.apache.doris.flink.sink.schema.SchemaChangeManager.dorisOptions of type org.apache.doris.flink.cfg.DorisOptions in instance of org.apache.flink.cdc.connectors.doris.sink.DorisSchemaChangeManager
```
1. 先移除`flink-cdc-pipeline-connector-doris-3.5.0.jar`
2. 重启集群
3. 再加入`flink-cdc-pipeline-connector-doris-3.5.0.jar`


mysql-to-doris.yml
```yml
################################################################################
# Description: Sync MySQL all tables to Doris
################################################################################
source:
   type: mysql
   hostname: 172.17.0.1
   port: 13306
   username: root
   password: "1234"
   tables: test.\.*
   server-id: 5400-5404
   server-time-zone: Asia/Shanghai

sink:
   type: doris
   fenodes: 172.17.0.1:8030
   username: root
   password: "1234"
   table.create.properties.light_schema_change: true
   table.create.properties.replication_num: 1

#route:
#   - source-table: app_db.orders
#     sink-table: ods_db.ods_orders
#   - source-table: app_db.shipments
#     sink-table: ods_db.ods_shipments
#   - source-table: app_db.products
#     sink-table: ods_db.ods_products

pipeline:
   name: Sync MySQL Database to Doris
   parallelism: 1

```

flink-cdc提交
```bash

# 方法1
## 1. 下载flink-1.20.5.tgz解压到本地,如/data/local/dinky/flink-1.20.5,注意额外的依赖也需放到lib下
## 2. 修改flink/conf/config.yml 加入:
# 远程集群的地址
rest.address: localhost 
# 远程集群的端口
rest.port: 18081
   
# 设置checkpoint提交, 否则不会自动同步
execution:
  checkpointing:
    interval: 30000 # 30秒                                                                                                                                                                                                                                                    
## 提交
./flink-cdc.sh --flink-home=/data/local/dinky/flink-1.20.5 --target remote mysql-doris.yml

# 方法2
## 把flink-cdc.tgz复制到集群的jobmanager里执行
docker compose exec jobmanager /opt/flink-cdc-3.5.0/bin/flink-cdc.sh /opt/flink-cdc-3.5.0/bin/mysql-doris.yml

```

dinky的自定义表同步:

```sql
set execution.checkpointing.interval = 10s;
set pipeline.operator-chaining=false;

CREATE TABLE IF NOT EXISTS mysql_x1 (
  id INT,
  name STRING,
  create_time TIMESTAMP
  ,PRIMARY KEY ( `id` ) NOT ENFORCED
) COMMENT 'x1'
 WITH (
  'connector' = 'mysql-cdc',
  'hostname' = '172.17.0.1',
  'port' = '13306',
  'username' = 'root',
  'password' = '1234',
  'database-name' = 'test',
  'table-name' = 'x1',
  'server-time-zone' = 'Asia/Shanghai',
  'jdbc.properties.tinyInt1isBit'='false'
);

CREATE TABLE IF NOT EXISTS doris_x1 (
  id INT,
  name STRING,
  create_time TIMESTAMP
  ,PRIMARY KEY ( `id` ) NOT ENFORCED
) COMMENT 'x1'
 WITH (
    'connector' = 'doris',
    'fenodes' = '172.17.0.1:8030',
    'table.identifier' = 'test.x1',
    'username' = 'root',
    'password' = '1234',
    -- 'sink.properties.format' = 'json',
    -- 'sink.properties.read_json_by_line' = 'true',
    'sink.enable-delete' = 'true',
    'sink.label-prefix' = 'test_x1_1'
);


insert into doris_x1 select * from mysql_x1;

```