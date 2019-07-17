## Hutool工具阅读记录

#### 转换

```java
//转换数组
public static void convertArray(){
    String[] b = { "1", "2", "3", "4" };
    //结果为Integer数组
    Integer[] intArray = Convert.toIntArray(b);
    logger.info("字符串数组转换Integer数组{}", Arrays.toString(intArray));

    long[] c = {1,2,3,4,5};
    //结果为Integer数组
    Integer[] intArray2 = Convert.toIntArray(c);
    logger.info("long数组转换Integer数组{}",Arrays.toString(intArray2));
}


//字符串转日期
public static  void convertToDate(){
    String str = "20190710";
    Date date = Convert.toDate(str);
    String str1 = "2019-07-11";
    Date date1 = Convert.toDate(str1);
    logger.info("日期转换输出{},{}",DateUtil.formatDate(date),DateUtil.formatDate(date1));
}

//数组转列表
public static void convertToList(){
    int[] nums = {1,2,3};
    List<?> numLists = Convert.toList(nums);
    numLists.stream().forEach(str->{
        logger.info(str+"");
    }
                             );
}

//编码转换
public static void charsetConvert(){
    String a = "字符串";
    String result = Convert.convertCharset(a, CharsetUtil.UTF_8, CharsetUtil.ISO_8859_1);
    String raw = Convert.convertCharset(result, CharsetUtil.ISO_8859_1, "UTF-8");
    logger.info("两个字符串是否相等{}",a.equals(raw));
}

//其它常用的转换
public static void otherConvert(){
    //16进制转换
    String a = "字符串";
    String hex = Convert.toHex(a, CharsetUtil.CHARSET_UTF_8);
    logger.info("字符串的16进制转换为{}",hex);

    //unicode转换
    String unicode = Convert.strToUnicode(a);
    logger.info("字符串的Unicode编码为{}",unicode);

    //结果为："陆万柒仟伍佰伍拾陆元叁角贰分"
    double money = 67556.32;
    String digitUppercase = Convert.digitToChinese(money);
    logger.info("转换为大写的金额{}",digitUppercase);

}
```

#### 日期

```java
//常见的获取当前时间的方法
public static void commonNowDateTool(){
    //当前时间
    Date date = DateUtil.date();
    //当前时间
    Date date2 = DateUtil.date(Calendar.getInstance());
    //当前时间
    Date date3 = DateUtil.date(System.currentTimeMillis());

    //前三个时间是等同的

    //当前时间字符串，格式：yyyy-MM-dd HH:mm:ss
    String now = DateUtil.now();
    //当前日期字符串，格式：yyyy-MM-dd
    String today= DateUtil.today();
}


//时间与日期的转换与字符串的获取
public static  void formatDate(){
    String dateStr = "2017-03-01";
    Date date = DateUtil.parse(dateStr);

    //常用格式的格式化，结果：2017-03-01
    String formatDate = DateUtil.formatDate(date);
    //结果：2017-03-01 00:00:00
    String formatDateTime = DateUtil.formatDateTime(date);
    //结果：00:00:00
    String formatTime = DateUtil.formatTime(date);

}

//获取一天当中的起始与终止
public static void dateStartAndEnd(){
    String dateStr = "2019-03-01 22:33:23";
    Date date = DateUtil.parse(dateStr);

    //一天的开始，结果：2019-03-01 00:00:00
    Date beginOfDay = DateUtil.beginOfDay(date);

    //一天的结束，结果：2019-03-01 23:59:59
    Date endOfDay = DateUtil.endOfDay(date);

}

//根据当前时间来获取上下时间范围的时间
public static void offestDate(){
    String dateStr = "2017-03-01 22:33:23";
    Date date = DateUtil.parse(dateStr);

    //结果：2017-03-03 22:33:23
    Date newDate = DateUtil.offset(date, DateField.DAY_OF_MONTH, 2);

    //常用偏移，结果：2017-03-04 22:33:23
    DateTime newDate2 = DateUtil.offsetDay(date, 3);

    //常用偏移，结果：2017-03-01 19:33:23
    DateTime newDate3 = DateUtil.offsetHour(date, -3);

    //昨天
    DateUtil.yesterday();
    //明天
    DateUtil.tomorrow();
    //上周
    DateUtil.lastWeek();
    //下周
    DateUtil.nextWeek();
    //上个月
    DateUtil.lastMonth();
    //下个月
    DateUtil.nextMonth();

}

//计算时间差
public static void computerDays(){
    String dateStr1 = "2017-03-01 22:33:23";
    Date date1 = DateUtil.parse(dateStr1);

    String dateStr2 = "2017-04-01 23:33:23";
    Date date2 = DateUtil.parse(dateStr2);

    //相差一个月，31天
    long betweenDay = DateUtil.between(date1, date2, DateUnit.DAY);

}


//其它常用日期工具方法
public static void otherDateTool(){
    //年龄
    DateUtil.ageOfNow("1990-01-30");

    //是否闰年
    DateUtil.isLeapYear(2017);
}
```

#### IO文件

```java
//IO的复制
public static void ioCopy(){
    BufferedInputStream in = FileUtil.getInputStream("d:/test.txt");
    BufferedOutputStream out = FileUtil.getOutputStream("d:/test2.txt");
    long copySize = IoUtil.copy(in, out, IoUtil.DEFAULT_BUFFER_SIZE);
}

//读取
public static void readFileLine() throws FileNotFoundException {
    //默认UTF-8编码，可以在构造中传入第二个参数做为编码
    FileReader fileReader = new FileReader("test.properties");
    List<String> result = fileReader.readLines();
}


//读取classpath的
public static void readClassPathFile() throws IOException {
    //使用工具类加载classPath的配置文件
    ClassPathResource resource = new ClassPathResource("db.properties");
    Properties properties = new Properties();
    properties.load(resource.getStream());
    logger.info(String.valueOf(properties));
}


//优化后的Dict
public static void dictTool(){
    Dict dict = Dict.create()
        .set("key1", 1)//int
        .set("key2", 1000L)//long
        .set("key3", DateTime.now());//Date

    Long v2 = dict.getLong("key2");//1000
    Date now = dict.getDate("key3");

    logger.info("时间值为{},long值为{}",DateUtil.formatDate(now),v2);

}


//读取csv文件,csv文件为每列以"|"隔开的文件
public static void csvReadTool(){
    CsvReader reader = CsvUtil.getReader();
    //从文件中读取CSV数据
    CsvData data = reader.read(FileUtil.file("test1.csv"));
    List<CsvRow> rows = data.getRows();
    //遍历行
    for (CsvRow csvRow : rows) {
        //getRawList返回一个List列表，列表的每一项为CSV中的一个单元格（既逗号分隔部分）
        Console.log(csvRow.getRawList());
    }

}
```

#### HTTP

```java
public class HttpRequestHelper {

    private static  Logger logger = LoggerFactory.getLogger(HttpRequestHelper.class);

    public static <T> T request(HttpRequest request, Object params, Class<T> responseClass) {
        logger.debug("http请求的param{}",params);

        try {
            HttpResponse response = request.form(BeanUtil.beanToMap(params)).execute();

            if (response == null) {
               logger.error("http请求失败,无返回");
                return null;
            }

            if (response.getStatus() != HttpStatus.HTTP_OK) {
                logger.error("http请求返回状态不正确,状态为{},返回信息为{}",response.getStatus(),response.body());
                return null;
            }

            if (response.body() == null) {
                logger.error("http请求返回返回信息为空");
                return null;
            }

            T result = JSON.parseObject(response.body(), responseClass);
            return result;
        } catch (Exception e) {
            logger.error("http请求发生异常",e);
            return null;
        }
    }


}
```













