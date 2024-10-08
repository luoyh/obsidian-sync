


# 功能说明

向上输出一切地图相关信息, 比如经纬度反查地址, 经纬度查询道路等级, 道路查询, 区域行政区查询, 周边范围查询等.
向下对接第三方地图厂商, 可配置化, 支持多个厂商, 不同公司支持自定义资源查询.


```mermaid
flowchart LR
    设备--tcp--> 808服务
    808服务--tcp--> 设备
    其他设备 --tcp--> 808转入服务
    808转入服务 --tcp--> 其他设备
    
    其他平台 --tcp--> 808转出服务
    808转出服务 --tcp--> 其他平台
    
    forward --http--> 808服务
    808服务 --http--> forward
    forward --http--> 808转入服务
    808转入服务 --http--> forward
    forward --http--> 808转出服务
    808转出服务 --http--> forward
    
    forward --http--> gateway
    gateway --http--> forward
    
    gateway --http--> 前端
    前端 --http--> gateway
   
    808服务 --jt808_http_product_topic--> kafka
    808转入服务 --jt808_http_product_topic--> kafka
    808转出服务 --jt808_http_product_topic--> kafka
    kafka --jt808_http_product_topic--> stream
    stream --websocket--> 前端
    
    808服务 --jt808_trajectory_topic--> kafka
    808转入服务 --jt808_trajectory_topic--> kafka
    809上级平台 --jt808_trajectory_topic--> kafka
    kafka --jt808_trajectory_topic--> flink
    flink --存储轨迹--> doris
    flink --t808MessageTest--> kafka
    kafka --t808MessageTest-->stream
    
    808服务 --jt808_device_online_topic--> kafka
    808转入服务 --jt808_device_online_topic--> kafka
    809上级平台 --jt808_device_online_topic--> kafka
    kafka --jt808_device_online_topic--> flink
    
    809下级平台 --tcp--> 809上级平台
    809上级平台 --tcp--> 809下级平台
    
    809下级平台 --jt809_slave_product_topic--> 808服务
    809下级平台 --jt809_slave_product_topic--> kafka
    kafka --jt809_slave_product_topic--> flink
    flink --jt809_slave_product_topic--> doirs
    
    809上级平台 --jt809_master_product_topic--> kafka
    kafka --jt809_master_product_topic--> flink
    flink --jt809_master_product_topic--> doirs
    
    808服务 --jt808_product_topic--> kafka
    kafka:::someclass --jt808_product_topic--> 809下级平台
    
    classDef someclass fill:#fcff33
    

```


```mermaid
sequenceDiagram
前端->>代理服务: http请求指令
activate 前端
activate 代理服务
代理服务 ->> 808服务器: 转发指令
activate 808服务器
808服务器->>设备: 发送指令
808服务器 ->> 代理服务: 返回调用结果
deactivate 808服务器
代理服务->>前端: 返回调用结果
deactivate 代理服务
deactivate 前端
设备-->>808服务器: 返回调用指令结果
activate 808服务器
808服务器-->>kafka: 发送调用结果
deactivate 808服务器
activate kafka
kafka -->> stream服务: 监听kafka获取结果
deactivate kafka
activate stream服务
stream服务 -->> 前端: 通过websocket将结果推送给前端
deactivate stream服务
```

