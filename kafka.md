
```bash

# 查看消费组消费情况
./kafka-consumer-groups.sh --describe  --bootstrap-server 127.0.0.1:9092 --group consumer-group-id

# 查看所有消费组
./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看所有topic
./kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看topic信息
./kafka-topics.sh --describe  --bootstrap-server 127.0.0.1:9092 --topic test-topic


bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic my_topic_name --partitions 40

```

## kakfa代理转发

kakfa代理, 比如在kafka的server.properties配置的是 
```properties
listeners=PLAINTEXT://:19092,CONTROLLER://:9093,EXTERNAL://:9094

# Listener name, hostname and port the broker will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=PLAINTEXT://kafka.test-common:19092,EXTERNAL://192.168.1.13:19092
```

那么在一个可以转发的机器(192.168.1.15)设置nginx如下:


```nginx
stream {
    server {
        listen 29092;
        proxy_pass testkafka;
    }
    upstream testkafka {
        server kafka.test-common:19092;
    }
}
```

设置hosts
```
192.168.1.13 kafka.test-common
```

这时在其他机器就可以使用`192.168.1.15:29092`连接kafka.

如果上述不行的话, 就需要在本地在启动一个nginx转发.

```nginx
stream {
    server {
        listen 19092 so_keepalive=on;
        proxy_pass 192.168.1.15:29092;
        proxy_timeout 2h;
    }
}
```

然后本地的hosts需要配置:

```
127.0.0.1 kafka.test-common
```

这时使用`kafka.test-comomon:19092`连接即可

```bash
# 监听某个topic的消费情况
lag=$(/data/kafka_2.13-3.8.0/bin/kafka-consumer-groups.sh --describe  --bootstrap-server 10.0.1.3:9092 --group test-consumer 2>&1  | grep my-topic | awk '
{
 print $0
 offset += $4
 end += $5
 remain += $6
} 
END {
 printf "current offset: %d \t end: %d \t lag: %d\n", offset, end, remain
}
')

echo "`date` current consumer:" >> lag.out
echo "$lag" >> lag.out
echo "" >> lag.out

```