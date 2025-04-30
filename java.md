

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

```java

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.nio.ByteBuffer;

import com.alibaba.fastjson2.JSON;

import cn.hutool.core.util.HexUtil;

/**
 *
 * @author luoyh(Roy) - Apr 30, 2025
 * @since 21
 */
public class RedisHexAsBytes {
    
    public static void main(String[] args) throws Exception {
        char[] cc = redis.toCharArray();
        System.out.println(cc.length);
        int s = cc.length;
        ByteBuffer bb = ByteBuffer.allocate(cc.length);
        int len = 0;
//        byte[] raw = new byte[3351];
        for (int i = 0; i < s; i ++) {
            char c = cc[i];
            System.out.println(c);
            if (i < s - 1 && c == '\\' && cc[i + 1] == 'x') {
                System.out.println(">> " + cc[i + 2] + "" + cc[i + 3]);
                bb.put((byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 3;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == 'r') {
                System.out.println(">> \\r");
                bb.put((byte) ('\r'));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == 'n') {
                System.out.println(">> \\n");
                bb.put((byte) ('\n'));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == 't') {
                System.out.println(">> \\t");
                bb.put((byte) ('\t'));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == 'b') {
                System.out.println(">> \\b");
                bb.put((byte) ('\b'));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == 'a') {
                System.out.println(">> \\a");
                bb.put((byte) 0x07);
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == '"') {
                System.out.println(">> \"");
                bb.put((byte) ('"'));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            if (i < s - 1 && c == '\\' && cc[i + 1] == '\\') {
                System.out.println(">> \\");
                bb.put((byte) ('\\'));
//                raw[len ++] = (byte) (Integer.parseInt(cc[i + 2] + "" + cc[i + 3], 16) & 0xff);
                i += 1;
                continue;
            }
            bb.put((byte) c);
            System.out.println(">> " + c);
//            raw[len ++] = (byte) c;
        }
        int size = bb.position();
        bb.flip();
        byte[] raw = new byte[size];
        bb.get(raw);
        
        System.out.println(raw.length + "," + len);
        System.out.println(HexUtil.encodeHexStr(raw));
        
//
        try (ObjectInputStream input = new ObjectInputStream(new ByteArrayInputStream(raw))) {
            System.out.println("input.available(): " + input.available());
//        System.out.println("input.readByte(): " + input.readByte());
            System.out.println(JSON.toJSONString(input.readObject()));
        }
    }
    
    private static final String redis = "\\xac\\xed\\x00\\x05sr\\x002";

}

```