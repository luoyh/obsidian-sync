
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