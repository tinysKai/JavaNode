## Java8温习

### Lambda

#### 语法

+ (parameter) -> statement
+ (parameter) -> {statement;}

#### 何处使用

在`函数式接口上`使用Lambda

> 函数式接口就是只定义一个抽象方法的接口

常见的函数式接口

```java
public interface Predicate<T>{
    boolean test (T t);
}


@FunctionalInterface
public interface Supplier<T> {
    T get();
}

@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}    

@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}



public interface Comparator<T> {
    int compare(T o1, T o2);
}
public interface Runnable{
    void run();
}

```

#### 例子

```java
public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        //使用匿名类来构造runnable
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Hello World");
            }
        });

        //使用lambda来构造runnable,方法签名与上述匿名函数一致
        Thread thread2 = new Thread(() -> System.out.println("Hello World2"));


        thread.start();
        thread2.start();

        Thread.sleep(1000L);

    }
}
```



### 方法引用

方法引用就是让我们根据已有的方法实现来创建lambda表达式

#### 例子

```java
public class HelloWorld {

    public static final String E = "e://";

    public static void main(String[] args) {
        out(listFileWithOldWay());

        System.out.println("---------------");

        out(listFileWithJava8());
    }

    private static File[] listFileWithOldWay() {
        return new File(E).listFiles(new FileFilter() { //匿名类,一种不声明类名的实现了接口的类
            public boolean accept(File file) {
                return file.isDirectory();
            }
        });
    }

    private static File[] listFileWithLambda() {
        return new File(E).listFiles(file -> file.isDirectory());
    }

    private static File[] listFileWithJava8() {
        return  new File(E).listFiles(File::isDirectory);
    }

    private static void out(File[] files) {
        for (File file : files) {
            System.out.println(file.getName());
        }
    }
}
```

```java
/**
 * 描述: 使用方法引用来简化lambda
 */
public class MethodReference {
    public static void main(String[] args) {
        Function<String,String> consumer = s -> s.toUpperCase();
        Function<String,String> referConsume = String::toUpperCase;

        out(consumer,"asd");
        out(referConsume,"qwe");

        System.out.println("------------");
        
        listSort();
    }

    public static void out(Function<String,String> function, String str){
        System.out.println(function.apply(str));
    }

    static  void listSort(){
        List<String> list = Arrays.asList("2","0","a","e","1");
        //lambda的实现方式如下
        //list.sort((o1, o2) -> o1.compareToIgnoreCase(o2));
        //方法引用的实现方式如下
        list.sort(String::compareToIgnoreCase);
        System.out.println(list);
    }
}
```





### 流

#### 定义

从支持数据处理操作的源生成的元素序列

#### 特性

流只能被消费一次

#### 流操作

+ 中间操作
+ 终结操作

#### 操作类型

![ea8dfeebeae8f05ae809ee61b3bf3094.jpg](http://ww1.sinaimg.cn/large/8bb38904gy1gc6lzrtmlkj21kk13yjvk.jpg)

#### 入门

```java
public class Stream {
    public static void main(String[] args) {
        helloworld();
    }

    private static void helloworld() {
        List<Integer> list = Arrays.asList(1,3,2,4,6,5);

        List<Integer> nums =  list.stream()
                .filter(n -> n >= 3)
                .sorted(Comparator.reverseOrder())
                .limit(3)
                .collect(Collectors.toList());


        System.out.println(nums);
    }
}

```



####  映射

```java
public class Stream {
    public static void main(String[] args) {
        mapStream();

        flatmapStream();
    }

    private static void mapStream() {
        User a = new User(16,"a");
        User b = new User(13,"b");
        User c = new User(18,"c");
        User d = new User(21,"d");

        List<User> users = Arrays.asList(a,b,c,d);
        users.stream()
                .filter(u -> u.getAge() > 13)
                .map(User::getName) //这里将user对象转换为String对象
                .sorted()           //升序排序
                .forEach(System.out::println);
    }

    /**
     * 将一个字符串List中的每个字符串的每个字符拆分出来后输出
     *
     * flatmap方法让你把一个流中的每个值都换成另一个流，然后把所有的流连接起来成为一个流
     */
    private static void flatmapStream(){
        List<String> list = Arrays.asList("Hello","World");
        list.stream()
                .map(s -> s.split("")) //如果木有下面的flatmap则将输出两个数组引用地址
                .flatMap(Arrays::stream)  //让每一个数组变成单独的一个流
                .forEach(System.out::println);
    }
}

```

#### 查找

```java
public class FindStream {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("a","b","c","eer","qwe");
        list.stream()
                .filter(s -> s.length() == 1)
                .findAny().ifPresent(System.out::println);


       String str =  list.stream()
                .filter(s -> s.length() == 3)
                .findAny().orElse("");
       System.out.println("寻找长度为3的任意一个字符串 : " + str);
    }
}
```

#### 归约

```java
public class ReduceStream {
    public static void main(String[] args) {
        sum();

        max();
    }


    /**
     * 求最大值
     */
    private static void max() {
        List<Integer> list = Arrays.asList(1,3,2,5);
        list.stream()
                 .reduce(Integer::max)
                .ifPresent(System.out::println);
    }


    /**
     * 使用reduce求和
     */
    private static void sum() {
        List<Integer> list = Arrays.asList(1,3,2,5);
        int val = list.stream()
                .reduce(0,(a,b) -> a + b);
        System.out.println(val);

        //使用方法引用的话如下
        int val2 = list.stream()
                .reduce(0,Integer::sum);
        System.out.println(val2);

        //省略初始累加值的0时如下
        List<Integer> list2 = new ArrayList<>();
        int val3 = list2.stream()
                .reduce(Integer::sum).orElse(0);
        System.out.println(val3);
    }
}

```

#### 数值流

```java
public class NumStream {
    public static void main(String[] args) {
        User a = new User(16,"a");
        User b = new User(13,"b");

        List<User> users = new ArrayList<>();
        users.add(a);
        users.add(b);

        int sum = users.stream()
                .mapToInt(User::getAge) //转换为int类型的int的流
                .sum();
        System.out.println(sum);
    }
}
```

#### 汇总

```java
public class CollectStream {
    public static void main(String[] args) {
        //groupby();

        //joining();

        collectAndThen();
    }

    /**
     * 转换函数返回的类型
     *
     * 包裹另一个收集器，对其结果应用转换函数
     */
    private static void collectAndThen() {
        List<String> names = Arrays.asList("tinys","zhangfei","lisan");
        int size = names.stream()
                .collect(Collectors.collectingAndThen(Collectors.toList(),List::size));
        System.out.println(size);
    }

    /**
     *  集合不能为null
     *  集合元素个数可以为0
     */
    private static void joining() {
        List<String> names = Arrays.asList("tinys","zhangfei","lisan");
        String str = names.stream()
                .collect(Collectors.joining(","));
        System.out.println(str);
    }

    /**
     * 按照对象的某个元素进行分组 -- 使用收集器的groupBy方法
     */
    private static void groupby() {
        User a = new User(16,"a");
        User b = new User(13,"b");
        User c = new User(15,"c");
        User d = new User(13,"d");
        User e = new User(16,"e");
        User f = new User(17,"f");

        List<User> users = Arrays.asList(a,b,c,d,e,f);

        //groupBy(f)其实是groupBy(f,toList)的简写
        Map<Integer,List<User>> maps = users.stream()
                .collect(Collectors.groupingBy(User::getAge)); //list按年龄转换为map

        for (Map.Entry<Integer, List<User>> integerListEntry : maps.entrySet()) {
            System.out.println(integerListEntry);
        }

        //转换为映射个数
        Map<Integer,Long> map = users.stream()
                .collect(Collectors.groupingBy(User::getAge,Collectors.counting()));

        for (Map.Entry<Integer, Long> m : map.entrySet()) {
            System.out.println(m);
        }

    }
}
```

### 默认方法

+ Java 8允许在接口内声明静态方法
+ Java 8引入了一个新功能，叫默认方法，通过默认方法你可以指定接口方法的默认实现

#### 接口默认实现冲突判断原则

1. 类或者父类中的方法优先级最高
2. 若类中无方法,是接口中方法重复,则子接口的优先级高
3. 若是相同等级的接口,则需显式调用,指明是哪一个接口的方法

### Optional

Optional类未实现Serializable接口,所以无法序列化.由于这个原因，如果你的应用使用了某些要求序列化的库或者框架，在域模型中使用Optional，有可能引发应用程序故障。

#### 基本方法

+ get  最简单但最不安全的方法
+ orElse(T other) 提供一个对象不存在时返回默认值的机制
+ orElseGet(Supplier<? extends T> other) 延迟调用版的orElse
+ orElseThrow
+ ifPresent(Consume<? extend T>)
+ isPresent()

#### 基本使用

```java
public class HelloWorld {
    public static void main(String[] args) {
        //声明一个空的Optional对象
        Optional<User> emptyUser = Optional.empty();

        //声明一个非空的对象
        User a = new User(16,"tinys");
        Optional<User> user = Optional.of(a);

        //可接受Null的Optional
        User b = null;
        Optional<User> nullUser = Optional.ofNullable(b);

        //获取包含对象的具体属性
        String name = user.map(User::getName).orElse("");
        System.out.println(name);

    }
}
```

### CompletableFuture

#### 提供的功能

+ 创建异步操作
+ 异步协同
+ 响应式操作
+ 错误处理

#### 串行

常见方法

+ thenApply
+ thenAccept
+ thenRun
+ thenCompose

```java
public class SerialTask {
    public static void main(String[] args) {

        //虽然1是异步的,但2和3是同步的,整体流程是1,2,3顺序执行,2依赖1的执行结果,3依赖2的执行结果
        //所以一般这种串行没必要这么写
        CompletableFuture<String> f0 =
                CompletableFuture.supplyAsync(
                        () -> "Hello World")      //①
                        .thenApply(s -> s + " QQ")  //②
                        .thenApply(String::toUpperCase);//③

        //输出结果 HELLO WORLD QQ
        System.out.println(f0.join());


    }
}
```

#### AND

常见方法

+ thenCombine
+ thenAcceptBoth
+ runAfterBoth

关系如下

![https://imgchr.com/i/G8y5tO](http://ww1.sinaimg.cn/large/8bb38904gy1gdemk1le0jj20vq0hbjyh.jpg)

```java
public class TeaTask {
    public static void main(String[] args) {

        //任务1：洗水壶->烧开水
        CompletableFuture<Void> f1 =
                CompletableFuture.runAsync(() -> {       //runAsync无返回结果
                    System.out.println("T1:洗水壶...");
                    sleep(1000L);
                    System.out.println("T1:烧开水...");
                    sleep(10000L);

                });

        //任务2：洗茶壶->洗茶杯->拿茶叶
        CompletableFuture<String> f2 =
                CompletableFuture.supplyAsync(() -> {   //supplyAsync有返回结果
                    System.out.println("T2:洗茶壶...");
                    sleep(1000L);

                    System.out.println("T2:洗茶杯...");
                    sleep(2000L);

                    System.out.println("T2:拿茶叶...");
                    sleep(1000L);
                    return "龙井";
                });
        //任务3：任务1和任务2完成后执行：泡茶
        CompletableFuture<String> f3 =
                f1.thenCombine(f2, (f1Result, f2Result) -> {
                    System.out.println("T1:拿到茶叶:" + f2Result);
                    System.out.println("T1:泡茶...");
                    return "上茶:" + f2Result;
                });
        //等待任务3执行结果
        System.out.println(f3.join());
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {}
     }

}
```

#### OR

主要方法

+ applyToEither
+ acceptEither 
+ runAfterEither

```java
/**
 * 描述: 模拟OR的并行关系
 * 2020-04-01
 */
public class ChooseTask {
    public static void main(String[] args) {

        CompletableFuture<String> f1 =
                CompletableFuture.supplyAsync(()->{
                    int t = new Random().nextInt(4);
                    sleep(t * 1000L);
                    System.out.println("one" + t);
                    return String.valueOf(t);
                });

        CompletableFuture<String> f2 =
                CompletableFuture.supplyAsync(()->{
                    int t = new Random().nextInt(5);
                    sleep(t * 1000L);
                    System.out.println("two " + t);
                    return String.valueOf(t);
                });

        //f1或f2先获取到的一个的值就返回给f3
        CompletableFuture<String> f3 =
                f1.applyToEither(f2,s -> s);

        System.out.println("end" + f3.join());
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {}
    }
}

```

#### 异常

主要方法

```java
CompletionStage exceptionally(fn);
CompletionStage<R> whenComplete(consumer);
CompletionStage<R> whenCompleteAsync(consumer);
CompletionStage<R> handle(fn);
CompletionStage<R> handleAsync(fn);
```

```java
public class ExceptionTask {
    public static void main(String[] args) {

        CompletableFuture<Integer>
                f0 = CompletableFuture
                .supplyAsync(()->7/0)
                 .thenApply(r->r*10)
                .exceptionally(e->0);

        System.out.println(f0.join());
    }
}
```

#### 独立配置线程池

```java
public class HelloWorld {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        //若有异常会带到返回的future中
        CompletableFuture<String> str = CompletableFuture.supplyAsync(()->{
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {}
            System.out.println("Hello World In Future");
            return "Hello";
        }, Executors.newSingleThreadExecutor());//这里只是说明可以在CompletableFuture支持传递一个线程池来执行任务
        System.out.println("Hello World");
        System.out.println(str.get());

    }
}
```

```java
public class CollaborationFuture {
    public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {
        //A,B两个线程协同
        CompletableFuture<Integer> num =  CompletableFuture.supplyAsync(()-> 2 )  //线程A
                .thenCombine(
                            CompletableFuture.supplyAsync(()-> 3), //线程B
                            (i,j)-> i*j  //将线程A与线程B得到的结果最终相乘
                            );
        System.out.println(num.get(1, TimeUnit.SECONDS));
    }
}
```










