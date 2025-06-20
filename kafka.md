
```bash

# 查看消费组消费情况
./kafka-consumer-groups.sh --describe  --bootstrap-server 127.0.0.1:9092 --group consumer-group-id

# 查看所有消费组
./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看所有topic
./kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看topic信息
./kafka-topics.sh --describe  --bootstrap-server 127.0.0.1:9092 --topic test-topic

# 修改partition
bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic my_topic_name --partitions 40

# 消费
bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic test-topic [--from-beginning]

# 消费指定partition
bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic test-topic --offset 1234 --partition 0

# 发送消息
bin/kafka-console-producer.sh  --bootstrap-server 127.0.0.1:9092 --topic test-topic
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


## kafka消费设置偏移量

```java

package com.tqbb.test;

import java.io.FileOutputStream;
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.Collections;
import java.util.List;
import java.util.Properties;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.Printed;
import org.apache.kafka.streams.processor.api.Processor;
import org.apache.kafka.streams.processor.api.ProcessorContext;
import org.apache.kafka.streams.processor.api.ProcessorSupplier;
import org.apache.kafka.streams.processor.api.Record;

import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.util.DateUtils;

import lombok.Data;
import lombok.SneakyThrows;
import lombok.experimental.Accessors;

/**
 *
 * @author luoyh(Roy) - Oct 11, 2024
 * @since 21
 */
public class KafkaTestGPSPrint3 {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-kafka-gps-consumer");
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "tqxing-kafka.tqxing-common:31514");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");

        final String outFile = "e:/.roy/tmp/gps/test-gps-%s.log".formatted(System.currentTimeMillis());
        gps1(props, outFile);
    }

    @SneakyThrows
    private static void gps1(Properties consumerProperties, final String outFile) {
        
        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProperties);
                FileOutputStream out = new FileOutputStream(outFile);
                ) {
            //List<TopicPartition> partitions = consumer.partitionsFor("jt808_trajectory_topic").stream().map(p -> new TopicPartition(p.topic(), 0)).toList();
            //consumer.assign(partitions);
            consumer.assign(Collections.singletonList(new TopicPartition("jt808_trajectory_topic", 0)));
            consumer.seek(new TopicPartition("jt808_trajectory_topic", 0), 759036165L);
//            consumer.subscribe(Collections.singletonList("jt808_trajectory_topic"));
            
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
                for (ConsumerRecord<String, String> rec : records) {
                    if (rec.value().indexOf("40934854390") > -1) {
                        out.write(("[" + DateUtils.format(rec.timestamp()) + "]:[" + rec.offset() + "]:" + rec.value() + System.lineSeparator()).getBytes());
                    }
                }
                out.flush();
                if (records.isEmpty()) {
                    Thread.sleep(Duration.ofSeconds(2));
                }
            }
        }
    }
    
    @SneakyThrows
    private static void gps(Properties consumerProperties, final String outFile) {
        
        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProperties);
                FileOutputStream out = new FileOutputStream(outFile);
                ) {
            consumer.subscribe(Collections.singletonList("jt808_trajectory_topic"));
            
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));
                for (ConsumerRecord<String, String> rec : records) {
                    if (rec.value().indexOf("40934854390") > -1) {
                        out.write(("[" + DateUtils.format(rec.timestamp()) + "]:[" + rec.offset() + "]:" + rec.value() + System.lineSeparator()).getBytes());
                    }
                }
                out.flush();
                if (records.isEmpty()) {
                    Thread.sleep(Duration.ofSeconds(2));
                }
            }
        }
    }
}

```