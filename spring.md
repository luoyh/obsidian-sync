
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