

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
publish rtws hello-world

# 订阅消息
subscribe rtws

# stream 读取消息
xread streams rtws 0-0

# stream 发布消息
xadd rtws maxlen = 10 * key msg
```


**spring data redis pub/sub/stream

```java

import java.time.Duration;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.stream.Consumer;
import org.springframework.data.redis.connection.stream.MapRecord;
import org.springframework.data.redis.connection.stream.ReadOffset;
import org.springframework.data.redis.connection.stream.StreamOffset;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.data.redis.stream.StreamListener;
import org.springframework.data.redis.stream.StreamMessageListenerContainer;
import org.springframework.data.redis.stream.StreamMessageListenerContainer.StreamMessageListenerContainerOptions;

import lombok.extern.slf4j.Slf4j;

/**
 *
 * @author luoyh(Roy) - Apr 7, 2025
 * @since 21
 */
@Slf4j
@Configuration
public class RedisConfigx implements CommandLineRunner {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private static final ExecutorService es = Executors.newFixedThreadPool(2);

//    @Bean
//    public MessageListenerAdapter redisMessageListener() {
//        return new MessageListenerAdapter(new RedisMessageListener());
//    }
//
//    @Bean
//    public RedisMessageListenerContainer redisContainer(RedisConnectionFactory factory) {
//        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
//        container.setConnectionFactory(factory);
//        container.addMessageListener(redisMessageListener(), redisTopic());
//        container.setTaskExecutor(es);
//        container.setErrorHandler(new ErrorHandler() {
//            
//            @Override
//            public void handleError(Throwable t) {
//                log.error("redis subscribe error", t);
//            }
//        });
//        container.setSubscriptionExecutor(es);
//        container.setRecoveryBackoff(null);
//
//        scheduler.scheduleAtFixedRate(() -> {
//            System.out.println("container.isActive()="+container.isActive());
//            System.out.println("container.isAutoStartup()="+container.isAutoStartup());
//            System.out.println("container.isListening()="+container.isListening());
//            System.out.println("container.isRunning()="+container.isRunning());
//            System.out.println("es.isShutdown()="+es.isShutdown());
//            System.out.println("es.isTerminated()="+es.isTerminated());
//            
//
//            redisTemplate.executePipelined((RedisCallback<Void>) act -> {
//                System.out.println("act.isSubscribed()="+act.isSubscribed());
//                return null;
//            });
//            
//        }, 30, 30, TimeUnit.SECONDS);
//        
//        return container;
//    }
    
    @Bean
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>> streamListener(RedisConnectionFactory factory) {
        StreamMessageListenerContainerOptions<String, MapRecord<String, String, String>> containerOptions = 
                StreamMessageListenerContainerOptions
                .builder()
                .pollTimeout(Duration.ofMillis(100))
//                .targetType(String.class)
                .build();
        

        StreamMessageListenerContainer<String, MapRecord<String, String, String>> container = 
                StreamMessageListenerContainer.create(factory, containerOptions);
        
//        StreamOffset<String> offset = StreamOffset.latest("rtws");
//        factory.getConnection()
//        .streamCommands()
//        .xGroupCreate(
//                offset.getKey().getBytes(), 
//                "g0", 
//                ReadOffset.latest(), 
//                true);
//        
        container.receive(StreamOffset.latest("rtws"), new StreamListener<String, MapRecord<String,String,String>>() {
            
            @Override
            public void onMessage(MapRecord<String, String, String> message) {
                log.info("on redis message: {}", message);
            }
        });
        
        
        container.start();
        return container;
    }
    
    @Bean
    public ChannelTopic redisTopic() {
        return new ChannelTopic("rtws");
    }

    @Override
    public void run(String... args) throws Exception {
//        redisTemplate.execute((RedisCallback<Void>) act -> {
//            act.subscribe(new RedisMessageListener(), "rtws".getBytes());
//            System.out.println(">>act.isSubscribed()="+act.isSubscribed());
//            return null;
//        }, false);
//
//        scheduler.scheduleAtFixedRate(() -> {
//            System.out.println("es.isShutdown()="+es.isShutdown());
//            System.out.println("es.isTerminated()="+es.isTerminated());
//            
//
//            redisTemplate.execute((RedisCallback<Void>) act -> {
//                System.out.println("act.isSubscribed()="+act.isSubscribed());
//                return null;
//            });
//            
//        }, 30, 30, TimeUnit.SECONDS);
    }

    
}

```


```java
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.stream.MapRecord;
import org.springframework.data.redis.connection.stream.StreamRecords;
import org.springframework.data.redis.connection.stream.StringRecord;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.stereotype.Component;

/**
 *
 * @author luoyh(Roy) - Apr 7, 2025
 * @since 21
 */
@Component
public class RedisMessagePublisher {
    
    @Autowired
    private ChannelTopic redisTopic;
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public void publish(String msg) {
//        redisTemplate.convertAndSend(redisTopic.getTopic(), msg);
//        redisTemplate.opsForValue().getOperations().convertAndSend(redisTopic.getTopic(), msg);
        Map<String, String> value = new HashMap<>();
        value.put("kkky", "value:::" + LocalDateTime.now().toString());
//        MapRecord<String, String, String> reco = MapRecord.create(null, null);
//        StreamRecords.newRecord().in("rtws").ofStrings(null);
        StringRecord rec = StringRecord.of(value).withStreamKey("rtws");
        redisTemplate.opsForStream().add(rec);
        redisTemplate.opsForStream().trim("rtws", 10);
    }
    
    

}

```