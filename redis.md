

```r
# 查看客户端
client list

# 关闭客户端
CLIENT KILL TYPE normal  # 关闭普通客户端
CLIENT KILL TYPE pubsub  # 关闭发布/订阅客户端
CLIENT KILL TYPE replica  # 关闭副本客户端
CLIENT KILL TYPE all  # 关闭所有类型的客户端

# 通过id关闭
CLIENT KILL ID 12345
# 通过ip端口
CLIENT KILL 127.0.0.1:63512


# 查看pub/sub的订阅数
# rtws 是channel
pubsub numsub rtws

# 发布消息
publish
```