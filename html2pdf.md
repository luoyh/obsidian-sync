
```xml
<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>html2pdf</artifactId>
  <version>6.0.0</version>
</dependency>
```

```java

import java.io.FileOutputStream;
import java.io.OutputStream;

import com.itextpdf.commons.datastructures.Tuple2;
import com.itextpdf.html2pdf.ConverterProperties;
import com.itextpdf.html2pdf.HtmlConverter;
import com.itextpdf.html2pdf.attach.ITagWorker;
import com.itextpdf.html2pdf.attach.ProcessorContext;
import com.itextpdf.html2pdf.attach.impl.OutlineHandler;
import com.itextpdf.html2pdf.attach.impl.TagOutlineMarkExtractor;
import com.itextpdf.html2pdf.html.TagConstants;
import com.itextpdf.kernel.pdf.PdfDictionary;
import com.itextpdf.kernel.pdf.PdfOutline;
import com.itextpdf.kernel.pdf.action.PdfAction;
import com.itextpdf.layout.font.FontProvider;
import com.itextpdf.styledxmlparser.node.IElementNode;

/**
 *
 * @author luoyh(Roy) - Dec 26, 2024
 * @since 21
 */
public class TestHtml2Pdf {

    public static void main(String[] args) throws Exception {
        final String pdfPath = "D:\\.work\\.tmp\\123\\html." + System.currentTimeMillis() + ".pdf";
        final String fontPath = "d:/chinese/simkai.ttf";
        try (OutputStream outputStream = new FileOutputStream(pdfPath);
                ) {
            ConverterProperties converterProperties = new ConverterProperties();
            FontProvider fontProvider = new FontProvider();
            fontProvider.addFont(fontPath);
            converterProperties.setFontProvider(fontProvider);
            // 支持书签
            // 或者使用OutlineHandler.createStandardHandler()
            converterProperties.setOutlineHandler(new ChangedOutlineHandler());
            HtmlConverter.convertToPdf("""
                    <!DOCTYPE html>
                    <html lang="en">
                     <head>
                      <meta charset="UTF-8">
                      <meta name="Generator" content="EditPlus®">
                      <meta name="Author" content="">
                      <meta name="Keywords" content="">
                      <meta name="Description" content="">
                      <title>Document</title>
                      <style>
                        body {
                            font-family: 'SimSun' !important;
                        }
                        table {
                            /*分页时表格换行, 可不用, 使用表格行换行即可*/
                            /*page-break-before: always;*/
                            /*border-collapse: collapse;*/
                            width: 30cm !important;
                            margin-right: 2pt; 
                            margin-left: 2pt; 
                            border: 1px solid #000; 
                            float: left; 
                        }
                        td {
                            border: 1px solid rgb(0, 0, 0); 
                            min-height: 40px;
                        }
                        tr {
                            /*分页时表格行换行*/
                            page-break-inside: avoid !important;
                            min-height: 30px;
                        }
                        /* 防止页面被截断 */
                        @page {  size: 329mm 297mm; }
                     </style>
                     </head>
                     <body>
                          <h1>1. 这是标签1 </h1>
                          <p> h1 content </p>
                          <h2>1.1 这是标签1.1 </h2>
                          <p> h2(1.1) content </p>
                          
                          <p style="page-break-before:always; clear:both;"></p>
                          <p>这是下一页的 this is next page</p>
                          <h1>1. 这是标签2 </h1>
                          <p> h1(2) content </p>
                     </body>
                    </html>

                    """, outputStream, converterProperties);
        }
        System.out.println("PDF created successfully.");
        
    }
    

    private static class ChangedOutlineHandler extends OutlineHandler {
        @Override
        protected OutlineHandler addOutlineAndDestToDocument(ITagWorker tagWorker, IElementNode element, ProcessorContext context) {
            String markName = markExtractor.getMark(element);
            if (null != tagWorker && hasMarkPriorityMapping(markName) && context.getPdfDocument() != null) {
                int level = (int) getMarkPriorityMapping(markName);
                if (null == currentOutline) {
                    currentOutline = context.getPdfDocument().getOutlines(false);
                }
                PdfOutline parent = currentOutline;
                while (!levelsInProcess.isEmpty() && level <= levelsInProcess.getFirst()) {
                    parent = parent.getParent();
                    levelsInProcess.pop();
                }
                PdfOutline outline = parent.addOutline(generateOutlineName(element));
                String destination = generateUniqueDestinationName(element);
                PdfAction action = PdfAction.createGoTo(destination);
                outline.addAction(action);
                destinationsInProcess.push(new Tuple2<String, PdfDictionary>(destination, action.getPdfObject()));

                levelsInProcess.push(level);
                currentOutline = outline;
            }
            return this;
        }
        public ChangedOutlineHandler(){
            markExtractor = new TagOutlineMarkExtractor();
            putMarkPriorityMapping(TagConstants.H1, 1);
            putMarkPriorityMapping(TagConstants.H2, 2);
            putMarkPriorityMapping(TagConstants.H3, 3);
            putMarkPriorityMapping(TagConstants.H4, 4);
            putMarkPriorityMapping(TagConstants.H5, 5);
            putMarkPriorityMapping(TagConstants.H6, 6);
        }
    }

}

```

## 支持toc

```html
<html>
  <body>

    <div class="toc-container">
      <div class="toc-row"><a href="#one">Link To Page One</a></div>
      <div class="toc-row"><a href="#two">Link To Page Two</a></div>
    </div>

    <div class="content-container">
      <div class="page" id="one">Page One Content Here</div>
      <div class="page" id="two">Page Two Content Here</div>
    </div>

  </body>
</html>
```

```css
.toc-row a::after {
  content: leader('.') target-counter(attr(href), page, decimal);
}
```

```java
package com.tqbb.test.pdf;

import java.io.File;
import java.io.FileOutputStream;
import java.io.OutputStream;
import java.util.UUID;

import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;
import org.jsoup.parser.Tag;
import org.jsoup.select.Elements;

import com.itextpdf.html2pdf.ConverterProperties;
import com.itextpdf.html2pdf.HtmlConverter;
import com.itextpdf.html2pdf.attach.impl.OutlineHandler;
import com.itextpdf.layout.font.FontProvider;

import lombok.SneakyThrows;

/**
 *
 * @author luoyh(Roy) - Jun 12, 2025
 * @since 21
 */
public class Itext2pdfTest002 {

    @SneakyThrows
    public static void main(String[] args) {
        try (OutputStream outputStream = new FileOutputStream(PDFCons.outpdf());
                //FileInputStream inputStream = new FileInputStream(new File(PDFCons.normal));
                
                ) {
            Document htmlDoc = Jsoup.parse(new File("D:/.work/.tmp/html/04.html"), "UTF-8");

            // This is our Table of Contents aggregating element
            Element tocElement = htmlDoc.select("#content").first().prependElement("div");
            //Element tocElement = htmlDoc.body().prependElement("div");
            tocElement.append("<p style=\"page-break-before:always;\"></p>");
            tocElement.append("<p style=\"text-align: center;\"><b>目录</b></p>");

            // We are going to build a complex CSS
            StringBuilder tocStyles = new StringBuilder().append("<style>");

            Elements tocElements = htmlDoc.select("H1,h2,h3,h4,h5,h6");
            int idx = 0;
            
            for (Element elem : tocElements) {
                int t = switch (elem.tagName()) {
                case "h1" -> 0;
                case "h2" -> 1;
                case "h3" -> 2;
                case "h4" -> 3;
                case "h5" -> 4;
                case "h6" -> 5;
                default -> 0;
                };
                // Here we create an anchor to be able to refer to this element when generating
                // page numbers and links
                String id = "_toc_h_" + (idx ++);//UUID.randomUUID().toString();
                elem.attr("id", id);

                // CSS selector to show page numebr for a TOC entry
                tocStyles.append("*[data-toc-id=\"" + id + "\"] .toc-page-ref::after { content: target-counter(#" + id
                        + ", page) }");

                // Generating TOC entry as a small table to align page numbers on the right
                Element tocEntry = tocElement.appendElement("table");
                tocEntry.attr("style", "width: 100%; border: none;");
                Element tocEntryRow = tocEntry.appendElement("tr");
                tocEntryRow.attr("data-toc-id", id);
                tocEntryRow.attr("style", "boder: none;");
                Element tocEntryTitle = tocEntryRow.appendElement("td");
                tocEntryTitle.attr("style", "border: none;");
                tocEntryTitle.append("<span style=\"margin-left: " + (40 * t) + "px;\">"+elem.text()+"</span>");
                Element tocEntryPageRef = tocEntryRow.appendElement("td");
                tocEntryPageRef.attr("style", "text-align: right; border: none;");
                // <span> is a placeholder element where target page number will be inserted
                // It is wrapped by an <a> tag to create links pointing to the element in our
                // document
                tocEntryPageRef.append("<a href=\"#" + id + "\"><span class=\"toc-page-ref\"></span></a>");
            }

            tocStyles.append("</style>");

            htmlDoc.head().append(tocStyles.toString());

            String html = htmlDoc.outerHtml();

            
//            
            System.out.println(html);
            
            ConverterProperties converterProperties = new ConverterProperties();
            FontProvider fontProvider = new FontProvider();
            fontProvider.addFont(PDFCons.fontPath);
            converterProperties.setFontProvider(fontProvider);
            converterProperties.setOutlineHandler(OutlineHandler.createStandardHandler());
//            converterProperties.setTagWorkerFactory(DefaultTagWorkerFactory.getInstance());
            System.out.println(converterProperties.getTagWorkerFactory());
//            converterProperties.setImmediateFlush(true);
//            converterProperties.setLimitOfLayouts(3);
            HtmlConverter.convertToPdf(html, outputStream, converterProperties);
        }
        System.out.println("PDF created successfully.");
    }

}


```


### html2转pdf, 支持书签与目录,并且目录支持跳转

`https://kb.itextpdf.com/itext/adding-bookmarks-table-of-contents-to-a-pdfhtml-co`

```java
package com.tqbb.test.pdf;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.ArrayDeque;
import java.util.Deque;

import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;
import org.jsoup.select.Elements;

import com.itextpdf.commons.datastructures.Tuple2;
import com.itextpdf.html2pdf.ConverterProperties;
import com.itextpdf.html2pdf.HtmlConverter;
import com.itextpdf.kernel.pdf.PdfDocument;
import com.itextpdf.kernel.pdf.PdfOutline;
import com.itextpdf.kernel.pdf.PdfWriter;
import com.itextpdf.kernel.pdf.action.PdfAction;
import com.itextpdf.layout.font.FontProvider;
import com.tqbb.test.util.IdWorker;

import lombok.SneakyThrows;

/**
 *
 * @author luoyh(Roy) - Jun 12, 2025
 * @since 21
 */
public class Itext2pdfTest3 {

    @SneakyThrows
    public static void main(String[] args) {

    try (
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            PdfDocument pdfDocument = new PdfDocument(new PdfWriter(
            //PDFCons.outpdf()
            out
            ))) {
        PdfOutline bookmarks = pdfDocument.getOutlines(false);
//        try (OutputStream outputStream = new FileOutputStream(PDFCons.outpdf());
//                //FileInputStream inputStream = new FileInputStream(new File(PDFCons.normal));
//                
//                ) {
            Document htmlDoc = Jsoup.parse(new File("D:/.work/.tmp/html/07.html"), "UTF-8");
            
    
             // This is our Table of Contents aggregating element
             Element tocElement = htmlDoc.body().prependElement("div");
             tocElement.append("<p style=\"page-break-before:always;\"></p>");
             tocElement.append("<h1 style=\"text-align: center;font-size:20px;\"><b>目录</b></h1>");
    
             // We are going to build a complex CSS
             StringBuilder tocStyles = new StringBuilder().append("<style>");
             Elements tocElements = htmlDoc.select("h1,h2,h3,h4,h5,h6");
             Deque<Tuple2<Integer, PdfOutline>> stack = new ArrayDeque<>();
             for (Element elem : tocElements) {
                 int t = switch (elem.tagName()) {
                 case "h1" -> 0;
                 case "h2" -> 1;
                 case "h3" -> 2;
                 case "h4" -> 3;
                 case "h5" -> 4;
                 case "h6" -> 5;
                 default -> 0;
                 };
                 // Here we create an anchor to be able to refer to this element when generating page numbers and links
                 String id = elem.attr("id");
                 if (id == null || id.isEmpty()) {
                     id = "x" + IdWorker.id();
                     elem.attr("id", id);
                 }

                 // CSS selector to show page numbers for a TOC entry
                 tocStyles.append("*[data-toc-id=\"").append(id)
                         .append("\"] .toc-page-ref::after { content: target-counter('#").append(id).append("', page) }");

                 // Generating TOC entry as a small table to align page numbers on the right
                 Element tocEntry = tocElement.appendElement("div");
                 tocEntry.attr("style", "width: 100%; border-bottom: 1px dashed rgb(200,200,200);text-align:left; ");
                 Element tocEntryRow = tocEntry.appendElement("a");
                 tocEntryRow.attr("data-toc-id", id);
                 tocEntryRow.attr("style", "boder: none;text-decoration:none; color: black;");
                 tocEntryRow.attr("href", "#" + id);
                 tocEntryRow.append("<span style=\"width:"+(20 * t)+"px;display: inline-block;\"></span>");
                 Element tocEntryTitle = tocEntryRow.appendElement("span");
//                 tocEntryTitle.attr("style", "padding-left:" + (10 * t) + "px;");
//                 tocEntryTitle.append("<span style=\"margin-left: " + (20 * t) + "px;\">"+elem.text()+"</span>");
                 tocEntryTitle.appendText(elem.text());
                 Element tocEntryPageRef = tocEntryRow.appendElement("span");
                 tocEntryPageRef.attr("style", "float:right;");
                 tocEntryPageRef.attr("class", "toc-page-ref");
                 
                 // <span> is a placeholder element where target page number will be inserted
                 // It is wrapped by an <a> tag to create links pointing to the element in our
                 // document
//                 tocEntryPageRef.append("<span class=\"toc-page-ref\" style=\"float:right;\"></span>");
                 
                 PdfOutline bookmark = null;
                 Tuple2<Integer, PdfOutline> poll = null;
                 while ((poll = stack.poll()) != null) {
                     if (poll.getFirst() < t) {
                         bookmark = poll.getSecond().addOutline(elem.text());
                         stack.push(poll);
                         break;
                     }
                 }
                 
                 if (null == bookmark) {
                     bookmark = bookmarks.addOutline(elem.text());
                 }
                 
                 stack.push(new Tuple2<Integer, PdfOutline>(t, bookmark));
                 

                 //PdfOutline bookmark = bookmarks.addOutline(elem.text());
                 bookmark.addAction(PdfAction.createGoTo(id));
             }
    
             tocStyles.append("</style>");
    
             htmlDoc.head().append(tocStyles.toString());
    
             String html = htmlDoc.outerHtml();
            
            System.out.println(html);
            
            
            ConverterProperties converterProperties = new ConverterProperties();
            FontProvider fontProvider = new FontProvider();
            fontProvider.addFont(PDFCons.fontPath);
            converterProperties.setFontProvider(fontProvider);
//            converterProperties.setOutlineHandler(OutlineHandler.createStandardHandler());
            converterProperties.setImmediateFlush(false);
//            converterProperties.setTagWorkerFactory(DefaultTagWorkerFactory.getInstance());
            System.out.println(converterProperties.getTagWorkerFactory());
//            converterProperties.setImmediateFlush(true);
//            converterProperties.setLimitOfLayouts(3);
            HtmlConverter.convertToPdf(html, pdfDocument, converterProperties);
            
            Files.write(Paths.get(PDFCons.outpdf()), out.toByteArray());
        }
        System.out.println("PDF created successfully.");
    }

}

```

### use selenium

```xml
<dependency>
  <groupId>org.seleniumhq.selenium</groupId>
  <artifactId>selenium-java</artifactId>
  <version>4.27.0</version>
</dependency>

<dependency>
  <groupId>org.seleniumhq.selenium</groupId>
  <artifactId>selenium-chrome-driver</artifactId>
  <version>4.27.0</version>
</dependency>
```

```java
package com.test;

import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

import org.openqa.selenium.Pdf;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.print.PrintOptions;


/**
 *
 * @author luoyh(Roy) - Dec 27, 2024
 * @since 21
 */
public class TestHtml2Pdf {
    
    public static void main(String[] args) throws Exception {
//        generatePdf(Paths.get("C:/Users/huitao/Desktop/Untitled4.html"), Paths.get("D:\\.work\\.tmp\\123\\md." + System.currentTimeMillis() + ".pdf"));
        asPdf(Paths.get("C:/Users/1/Desktop/Untitled4.html"), Paths.get("D:\\.work\\.tmp\\123\\md." + System.currentTimeMillis() + ".pdf"));
    }
    
    private static void asPdf(Path inputPath, Path outputPath) throws IOException {
//        System.setProperty("webdriver.chrome.driver", "D:\\.work\\data\\chrome\\chromedriver-win64\\chromedriver.exe");
        ChromeOptions options = new ChromeOptions();
        options.setBinary("D:\\.work\\data\\chrome\\chrome-headless-shell-win64\\chrome-headless-shell.exe");
//        options.addArguments("--headless", "--disable-gpu", "--run-all-compositor-stages-before-draw");

        System.out.println(5);
        ChromeDriver chromeDriver = new ChromeDriver(options);
        System.out.println(4);
        chromeDriver.get("data:text/html;base64," + Base64.getEncoder().encodeToString(Files.readAllBytes(inputPath)));
        //chromeDriver.get(inputPath.toString());
        
        System.out.println(3);
        PrintOptions print = new PrintOptions();
        System.out.println(2);
        Pdf pdf = chromeDriver.print(print);
        System.out.println(1);
        Files.write(outputPath, Base64.getDecoder().decode(pdf.getContent()));
        chromeDriver.quit();
        System.out.println(0);
    }

    private static void generatePdf(Path inputPath, Path outputPath) throws Exception {
        try {
            System.setProperty("webdriver.chrome.driver", "D:\\.work\\data\\chrome\\chromedriver-win64\\chromedriver.exe");
            ChromeOptions options = new ChromeOptions();
            options.addArguments("--headless", "--disable-gpu", "--run-all-compositor-stages-before-draw");
            ChromeDriver chromeDriver = new ChromeDriver(options);
            chromeDriver.get(inputPath.toString());
            Map<String, Object> params = new HashMap();
            System.out.println("11111111111");
            String command = "Page.printToPDF";
            
            Map<String, Object> output = chromeDriver.executeCdpCommand(command, params);

            try {
                FileOutputStream fileOutputStream = new FileOutputStream(outputPath.toString());
                byte[] byteArray = java.util.Base64.getDecoder().decode((String) output.get("data"));
                fileOutputStream.write(byteArray);
                fileOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            chromeDriver.quit();
        } catch (Exception e) {
            e.printStackTrace(System.err);
            throw e;
        }
    }

}

```