

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
> mvn depencency:get
```

## 获取指定时间的前一个时间

```java
// 上一个周一, 当前时间是周一就返回当前时间
LocalDate.now().with(TemporalAdjusters.previousOrSame(DayOfWeek.MONDAY));
// 上一个
LocalDate.now().with(TemporalAdjusters.previous(DayOfWeek.MONDAY));
// 下一个周一
LocalDate.now().with(TemporalAdjusters.nextOrSame(DayOfWeek.MONDAY));

// 0点
LocalDateTime.now().truncatedTo(ChronoUnit.DAYS);
LocalDate.now().atStartOfDay();
LocalDate.now().atTime(LocalTime.MIN);

// 23:59:59
LocalDate.now().atTime(LocalTime.MAX);

// 0分
LocalDateTime.now().truncatedTo(ChronoUnit.HOURS);

```