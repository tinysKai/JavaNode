## POI处理大Excel文件

#### 使用Hutool工具类

```xml
 <dependencies>
     	<!--poi版本必须是3.17以上-->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>3.17</version>
        </dependency>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>3.17</version>
        </dependency>
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml-schemas</artifactId>
            <version>3.17</version>
        </dependency>

        <dependency>
            <groupId>xerces</groupId>
            <artifactId>xercesImpl</artifactId>
            <version>2.11.0</version>
        </dependency>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>4.6.1</version>
        </dependency>

    </dependencies>
```



```java
import cn.hutool.core.lang.Console;
import cn.hutool.poi.excel.ExcelUtil;
import cn.hutool.poi.excel.sax.handler.RowHandler;

import java.util.List;

/**
 * 描述 : 使用hutool直接使用sax来读取数据
 * 2019/8/15
 */
public class HutoolExcelReadBigData {
    public static void main(String[] args) {
        ExcelUtil.readBySax("f://a.xlsx", 0, createRowHandler());
    }

    private static RowHandler createRowHandler() {
        return new RowHandler() {
            public void handle(int sheetIndex, int rowIndex, List<Object> rowlist) {
                Console.log("[{}] [{}] {}", sheetIndex, rowIndex, rowlist);
            }
        };
    }
}
```





#### POI源码

```java
import org.apache.poi.openxml4j.exceptions.OpenXML4JException;
import org.apache.poi.openxml4j.opc.OPCPackage;
import org.apache.poi.openxml4j.opc.PackageAccess;
import org.apache.poi.ss.usermodel.DataFormatter;
import org.apache.poi.ss.util.CellAddress;
import org.apache.poi.ss.util.CellReference;
import org.apache.poi.util.SAXHelper;
import org.apache.poi.xssf.eventusermodel.ReadOnlySharedStringsTable;
import org.apache.poi.xssf.eventusermodel.XSSFReader;
import org.apache.poi.xssf.eventusermodel.XSSFSheetXMLHandler;
import org.apache.poi.xssf.eventusermodel.XSSFSheetXMLHandler.SheetContentsHandler;
import org.apache.poi.xssf.model.StylesTable;
import org.apache.poi.xssf.usermodel.XSSFComment;
import org.xml.sax.ContentHandler;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;
import org.xml.sax.XMLReader;

import javax.xml.parsers.ParserConfigurationException;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;


/**
 * 一个基本的XLSX - > CSV处理器
 *
 * 使用SAX解析器读取数据表以保持内存占用比较小，所以这应该是能够阅读庞大的工作簿。
 * 样式表和共享字符串表必须保存在内存中。该标准POI样式表类使用，但是一个自定义（只读）类用于共享字符串表
 * 因为标准的POI SharedStringsTable增长很大快速与唯一字符串的数量。
 * <p/>
 */
public class ExcelReadBigData {
    /**
     * 用来在遍历时保存每一行的数据,遍历完会清空
     */
    private StringBuilder sb = new StringBuilder();

    /**
     * 保存数据的容器,数据量在10W以下时可直接使用ArrayList,超过这个数值时需注意切换为LinkedList比较安全
     */
    List<String> list = new ArrayList<>(512);



    /**
     * 使用XSSF Event SAX助手进行大部分工作
     * 解析Sheet XML，并输出内容
     * 作为（基本）CSV。
     */
    private class SheetToCSV implements SheetContentsHandler {
        private boolean firstCellOfRow = false;
        private int currentRow = -1;
        private int currentCol = -1;

        /**
         * 输出缺失的行
         *
         * @param number
         */
        private void missingRows(int number) {
            for (int i = 0; i < number; i++) {
                for (int j = 0; j < minColumns; j++) {
                    sb.append(',');
                }
                sb.append('\n');
            }
        }

        @Override
        public void startRow(int rowNum) {
            // If there were gaps, sb the missing rows
             missingRows(rowNum - currentRow - 1);
            // Prepare for this row
            firstCellOfRow = true;
            currentRow = rowNum;
            currentCol = -1;
        }

        @Override
        public void endRow(int rowNum) {
            // Ensure the minimum number of columns
            for (int i = currentCol; i < minColumns; i++) {
                sb.append(',');
            }
            sb.append('\n');

            if (currentRow != 0){
                //添加到list中
                list.add(sb.toString());
            }
            //清空字符串緩存
            sb.delete(0,sb.length());
        }

        @Override
        public void cell(String cellReference, String formattedValue, XSSFComment comment) {
            if (firstCellOfRow) {
                firstCellOfRow = false;
            } else {
                sb.append(',');
            }

            // gracefully handle missing CellRef here in a similar way as XSSFCell does
            if (cellReference == null) {
                cellReference = new CellAddress(currentRow, currentCol).formatAsString();
            }

            // Did we miss any cells?
            int thisCol = (new CellReference(cellReference)).getCol();
            int missedCols = thisCol - currentCol - 1;
            for (int i = 0; i < missedCols; i++) {
               sb.append(',');
            }
            currentCol = thisCol;
            //将标题单独处理
            if(currentRow == 0){
                //标题
                System.out.print(formattedValue + "     ");
            }else{
                // Number or string?
                try {
                    Double.parseDouble(formattedValue);
                    sb.append(formattedValue);
                } catch (NumberFormatException e) {
                    sb.append('"');
                    sb.append(formattedValue);
                    sb.append('"');
                }
            }

        }

        @Override
        public void headerFooter(String text, boolean isHeader, String tagName) {
            // Skip, no headers or footers in CSV
        }
    }


    /**
     * 表示可以存储多个数据对象的容器。
     */
    private final OPCPackage xlsxPackage;

    /**
     * 以最左边开始读取的列数
     */
    private final int minColumns;


    /**
     * 创建一个新的XLSX -> CSV转换器
     *
     * @param pkg        The XLSX package to process
     * @param minColumns 要输出的最小列数，或-1表示最小值
     */
    public ExcelReadBigData(OPCPackage pkg, int minColumns) {
        this.xlsxPackage = pkg;
        this.minColumns = minColumns;
    }

    /**
     * Parses and shows the content of one sheet
     * using the specified styles and shared-strings tables.
     * 解析并显示一个工作簿的内容
     * 使用指定的样式和共享字符串表。
     *
     * @param styles           工作簿中所有工作表共享的样式表。
     * @param strings          这是处理共享字符串表的轻量级方式。 大多数文本单元格将引用这里的内容。请注意，如果字符串由不同格式的位组成，则每个SI条目都可以有多个T元素
     * @param sheetInputStream
     */
    public void processSheet(StylesTable styles,
                             ReadOnlySharedStringsTable strings,
                             SheetContentsHandler sheetHandler, InputStream sheetInputStream)
            throws IOException, ParserConfigurationException, SAXException {

        DataFormatter formatter = new DataFormatter();
        InputSource sheetSource = new InputSource(sheetInputStream);
        try {
            XMLReader sheetParser = SAXHelper.newXMLReader();
            ContentHandler handler = new XSSFSheetXMLHandler(styles, null,
                    strings, sheetHandler, formatter, false);
            sheetParser.setContentHandler(handler);
            sheetParser.parse(sheetSource);
        } catch (ParserConfigurationException e) {
            throw new RuntimeException("SAX parser appears to be broken - "
                    + e.getMessage());
        }
    }

    /**
     * Initiates the processing of the XLS workbook file to CSV.
     * 启动将XLS工作簿文件处理为CSV。
     */
    public void process() throws IOException, OpenXML4JException, ParserConfigurationException, SAXException {
        ReadOnlySharedStringsTable strings = new ReadOnlySharedStringsTable(this.xlsxPackage);
        XSSFReader xssfReader = new XSSFReader(this.xlsxPackage);
        StylesTable styles = xssfReader.getStylesTable();
        XSSFReader.SheetIterator iter = (XSSFReader.SheetIterator) xssfReader.getSheetsData();
        int index = 0;
        while (iter.hasNext()) {
            InputStream stream = iter.next();
            String sheetName = iter.getSheetName();
            System.out.println(sheetName + " [index=" + index + "]:");
            processSheet(styles, strings, new SheetToCSV(), stream);
            stream.close();
            ++index;
        }
    }

    public static void main(String[] args) throws Exception {

        File xlsxFile = new File("f:/a.xlsx");
        if (!xlsxFile.exists()) {
            System.err.println("没找到文件: " + xlsxFile.getPath());
            return;
        }

        int minColumns = -1;
        if (args.length >= 2)
            minColumns = Integer.parseInt(args[1]);

        // The package open is instantaneous, as it should be.
        OPCPackage p = OPCPackage.open(xlsxFile.getPath(), PackageAccess.READ);
        ExcelReadBigData readBigData = new ExcelReadBigData(p,minColumns);
        readBigData.process();
        p.close();

        System.out.println();
        //读取完数据在list中
        for (String s : readBigData.list) {
            System.out.print(s);
        }

        //使用完清除记录释放jvm内存
        readBigData.list.clear();
    }
}
```

