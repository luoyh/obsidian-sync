
## endpoint开启或关闭日志

```bash
# debug
curl -i -X POST -H 'Content-Type: application/json' \
    -d '{"configuredLevel": "DEBUG"}'  \
    http://localhost:4001/actuator/loggers/com.tqbb.tuqiaoxing.admin.biz.mapper

# info
curl -i -X POST -H 'Content-Type: application/json' \
    -d '{"configuredLevel": "INFO"}' \
    http://localhost:4001/actuator/loggers/com.tqbb.tuqiaoxing.admin.biz.mapper



docker logs -f --tail 100 tuqiaoxing-admin-biz


```


### spring-doc (swagger)

```yml
server:
  port: 4001
spring:
  application:
    name: admin-biz-luoyh
  servlet:
    multipart:
      enabled: true
      max-file-size: 1000MB
      max-request-size: 1000MB
  cache:
    type: redis
  data:
    redis:
      password: pwd@123
      port: 6193
      host: localhost
      connect-timeout: 2000
      #lettuce.pool:
      #  enabled: true
      #  min-idle: 1
      # database: 13
  cloud:
    nacos:
      discovery.enabled: false
      config:
        enabled: false
        import-check.enabled: false
    sentinel:
      eager: true
      transport:
        dashboard: tuqiaoxing-sentinel:5003
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: pwd@123
    url: jdbc:mysql://localhost:3306/db01?allowMultiQueries=true&serverTimezone=Asia/Shanghai&nullCatalogMeansCurrent=true
    hikari:
      connection-timeout: 30000
      maximumPoolSize: 2
      minIdle: 2
      minimumIdle: 2
      keepaliveTime: 120000
      validationTimeout: 800

ffmpeg.path: /x
logging:
  level:
    root: info
    
tqbb.sysfile.font.skip: true
tqbb.pdf.html2pdf.disabled: true
# tqbb.pdf.enabled: false
# tqbb.pdf.html2pdf.disabled: false

springdoc:
  use-fqn: true
  swagger-ui:
    path: /swagger-ui.html
    tags-sorter: alpha
    operations-sorter: alpha
  api-docs:
    path: /v3/api-docs
  group-configs:
    - group: '误报'
      paths-to-match: '/alarm/**'
      packages-to-scan: com.xx.controller
    - group: '报警规则'
      paths-to-match: '/rule/**'
      packages-to-scan: com.xx.controller
      

# 暴露监控端点
management:
  endpoints:
    web:
      exposure:
        include: "*"  
  endpoint:
    health:
      show-details: ALWAYS


# feign 配置
feign:
  sentinel:
    enabled: true
  okhttp:
    enabled: true
  httpclient:
    enabled: false
  client:
    config:
      default:
        connectTimeout: 10000
        readTimeout: 10000
  compression:
    request:
      enabled: true
    response:
      enabled: true

# mybaits-plus配置
mybatis-plus:
  mapper-locations: classpath:/mapper/*Mapper.xml
  global-config:
    banner: false
    db-config:
      id-type: auto
      table-underline: true
      logic-delete-value: 1
      logic-not-delete-value: 0
  configuration:
    map-underscore-to-camel-case: true

```