

```bash

# 使用java运行jar,指定cp路径
# 比如依赖都在lib目录下
$JDK23/java -cp $(find ./lib/ -name  "*.jar" | xargs | sed  "s/ /:/g") -jar all-0.0.1-SNAPSHOT.jar arg0 arg1


```