

```bash

# 使用java运行jar,指定cp路径
# 比如依赖都在lib目录下
> $JDK23/java -cp .:$(find ./lib/ -name  "*.jar" | xargs | sed  "s/ /:/g") -jar all-0.0.1-SNAPSHOT.jar arg0 arg1
> $JDK22/java -cp .:$(find ./lib/ -name  "*.jar" | xargs | sed  "s/ /:/g") SimpleSender [...args]
> $JDK22/javac -cp $(find ./lib/ -name  "*.jar" | xargs | sed  "s/ /:/g") SimpleSender.java

# mvn install jar to local 
> mvn install:install-file -DgroupId=com.oracle -DartifactId=ojdbc14 -Dversion=10.2.0.2.0 -Dpackaging=jar -Dfile=ojdbc14-10.2.0.2.0.jar

# mvn copy dependency 
> mvn dependency:copy-dependencies

> mvn depencency:tree
> 
```