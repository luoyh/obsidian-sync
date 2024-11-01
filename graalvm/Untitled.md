
```bash
# a simple java class to executable
> ../native-image --enable-http --enable-https \
> -cp .:jsoup-1.18.1.jar Dict

# a jar to executable
> native-image [options] -jar jarfile [executable name]

```


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
    }

}

```