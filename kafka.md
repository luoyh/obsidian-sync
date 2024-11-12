
```bash

# 查看消费组消费情况
./kafka-consumer-groups.sh --describe  --bootstrap-server 127.0.0.1:9092 --group consumer-group-id

# 查看所有消费组
./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看所有topic
./kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看topic信息
./kafka-topics.sh --describe  --bootstrap-server 127.0.0.1:9092 --topic test-topic
```