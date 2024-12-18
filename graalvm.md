
```bash
# a simple java class to executable
> ../native-image --enable-http --enable-https \
> -cp .:jsoup-1.18.1.jar Dict

# a jar to executable
> native-image [options] -jar jarfile [executable name]

```


## spring-boot to executable

```xml
    <build>
        <plugins>
            <plugin>
                 <groupId>org.graalvm.buildtools</groupId>
                 <artifactId>native-maven-plugin</artifactId>
             </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                     <image>
                       <buildpacks>
                           <buildpack>gcr.io/paketo-buildpacks/graalvm</buildpack>
                           <buildpack>gcr.io/paketo-buildpacks/java-native-image</buildpack>
                       </buildpacks>
                       <env>
                           <BP_JVM_VERSION>21.0.2</BP_JVM_VERSION>
                           <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                       </env>
                    </image>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>


```

## build

```bash

# and use cmd
mvn -Pnative native:compile -Dmaven.test.skip=true

```


## Dict as executable

```java

import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.select.Elements;

/**
 *
 * @author luoyh(Roy) - Oct 15, 2024
 * @since 21
 */
public class Dict {

    public static void main(String[] args) throws Exception {
        Document doc = Jsoup.connect("https://dict.cn/search?q=%s".formatted(args[0])).get();
        Elements word = doc.select("#content div.main div.word");
        System.out.println("\033[1m" + word.select("div.word-cont h1.keyword").text() + " \033[0m");
//        Elements phonetic = word.select("div.phonetic span");
//        for (Element ph : phonetic) {
//            System.out.println(ph.text() + " " + ph.select("bdo.EN-US").text());
//        }
        System.out.println(word.select("div.phonetic span").text() + "");
        Elements dict = word.select("div.basic.clearfix ul.dict-basic-ul li");
        if (dict.size() > 0) {
            dict.forEach(e -> System.out.println(e.text()));
        } else {
//            word.select("div.basic.clearfix ul.dict-basic-ul li").forEach(e -> System.out.println(e.text()));
            word.select("div.basic.clearfix ul li").forEach(e -> System.out.println(e.select("strong").text()));
        }
        doc.select("#content div.main div.layout.cn ul li").forEach(e -> System.out.println(e.text()));
        doc.select("#content div.main div.word div.shape").forEach(e -> System.out.println(e.text()));
    }

}

```