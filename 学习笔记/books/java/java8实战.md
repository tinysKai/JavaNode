《java8实战》读书笔记
=

### java8初探

#### 方法引用
```
//java8前的文件筛选
File[] hiddenFiles = new File(".").listFiles(new FileFilter() {
    public boolean accept(File file) {
        return file.isHidden();
    }
});

  
  
  
//java8的文件筛选
File[] hiddenFiles = new File(".").listFiles(File::isHidden);
  
//我的理解为将File的对象方法的运行结果当做上面匿名回调函数的返回值
```

```
//java8之前对一个list筛选的代码
public static List<Apple> filterGreenApples(List<Apple> apples){
    List<Apple> result = new ArrayList<>();
    for (Apple apple: apples){
        if ("green".equals(apple.getColor())) {
            result.add(apple);
        }
    }
    return result;
}
  
  
//java8
先在Apple对象声明一个判断是否是绿苹果的方法
public static boolean isGreenApple(Apple apple) {
    return "green".equals(apple.getColor());
}

//然后在新代码中对他进行引用
static List<Apple> filterApples(List<Apple> apples , Predicate<Apple> p) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple: apples){
        if (p.test(apple)) {
            result.add(apple);
        }
    }
    return result;
}
  
//调用代码
filterApples(apples,Apple::isGreenApple);
  
//Predicates是java8的一个接口,也有用Function<Apple,Boolean>

public interface Predicate<T>{
    boolean test(T t);
}
```


####  Lambda表达式
如果你不想为某个只使用一两次的方法定义对象方法,那么java8还提供了Lambda表达式
```
filterApples(inventory, (Apple a) -> "green".equals(a.getColor()) );
```


#### 流
```
//java8前对一个列表进行筛选并按照货币类型进行分类
Map<Currency, List<Transaction>> transactionsByCurrencies = new HashMap<>();
for (Transaction transaction : transactions) {
    if(transaction.getPrice() > 1000){
        Currency currency = transaction.getCurrency();
        List<Transaction> transactionsForCurrency = transactionsByCurrencies.get(currency);
        if (transactionsForCurrency == null) {
            transactionsForCurrency = new ArrayList<>();
            transactionsByCurrencies.put(currency,transactionsForCurrency);
        }
        transactionsForCurrency.add(transaction);
    }
  
  
//java8使用流的筛选
  Map<Currency, List<Transaction>> transactionsByCurrencies =
      transactions.stream()
                  .filter((Transaction t) -> t.getPrice() > 1000)
                  .collect(groupingBy(Transaction::getCurrency));

```


### Lambda

#### Lambda结构
![lambda](https://github.com/tinysKai/Note/blob/master/image/article/2018/0709/lambda.png)

```
//匿名函数
Comparator<Apple> byWeight = new Comparator<Apple>() {
    public int compare(Apple a1, Apple a2){
        return a1.getWeight().compareTo(a2.getWeight());
    }
};  
  
//Lambda表达式
Comparator<Apple> byWeight =
    (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());

```

#### Lambda基本语法
(parameters) -> expression  
   或  
(parameters) -> { statements; }

#### 在何处使用Lambda
在函数式接口上可以使用Lambda表达式,函数式接口就是只定义一个抽象方法的接口
```
//Lambda表达式
Runnable r1 = () -> System.out.println("Hello World 1");
//匿名内部类
Runnable r2 = new Runnable(){
    public void run(){
        System.out.println("Hello World 2");
    }
};
public static void process(Runnable r){    
    r.run();
}
process(r1);
process(r2);
//Lambda表达式
process(() -> System.out.println("Hello World 3"));

```

#### java8中常见函数式接口
+ Predicate
+ Consumer
+ Function

```java
@FunctionalInterface
public interface Predicate<T>{
    boolean test(T t);
}
  

@FunctionalInterface
public interface Consumer<T>{
    void accept(T t);
}

 
@FunctionalInterface
public interface Function<T, R>{
    R apply(T t);
}

```

#### 类型推断
```
Comparator<Apple> c =
    (a1, a2) -> a1.getWeight().compareTo(a2.getWeight());

```

#### 方法引用
```
//对list进行排序 
  

//Lambda
List<String> str = Arrays.asList("a","b","A","B");
str.sort((s1, s2) -> s1.compareToIgnoreCase(s2));

//方法引用
List<String> str = Arrays.asList("a","b","A","B");
str.sort(String::compareToIgnoreCase);

```


### Stream流

#### 流初识
集合储存数据,流处理数据
```
//java7对菜单的筛选
List<Dish> lowCaloricDishes = new ArrayList<>();
for(Dish d: menu){
    if(d.getCalories() < 400){
        lowCaloricDishes.add(d);
    }                                                               
}                                                                  
Collections.sort(lowCaloricDishes, new Comparator<Dish>() {
    public int compare(Dish d1, Dish d2){
        return Integer.compare(d1.getCalories(), d2.getCalories());
    }
});
List<String> lowCaloricDishesName = new ArrayList<>();
for(Dish d: lowCaloricDishes){
    lowCaloricDishesName.add(d.getName());
}
  
//java8对菜单的筛选
 List<String> lowCaloricDishesName =
                   menu.stream()
                       .filter(d -> d.getCalories() < 400)
                       .sorted(comparing(Dish::getCalories))
                       .map(Dish::getName)
                       .collect(toList());


```

#### 流的组成
+ 一个数据源（如集合）来执行一个查询
+ 一个中间操作链，形成一条流的流水线
+ 一个终端操作，执行流水线，并能生成结果

#### 常见流操作
中间操作
+ filter
+ map
+ flatMap
+ sorted
+ limit
+ distinct
+ skip

终端操作
+ forEach
+ count
+ collect
+ reduce

```
//一个map与flatMap的对比例子,
 public void testMapAndFlatMap() {
        List<String> words = new ArrayList<String>();
        words.add("hello");
        words.add("word");
 
        //将words数组中的元素再按照字符拆分，然后字符去重，最终达到["h", "e", "l", "o", "w", "r", "d"]
        //如果使用map，是达不到直接转化成List<String>的结果
        List<String> stringList = words.stream()
                .flatMap(word -> Arrays.stream(word.split("")))
                .distinct()
                .collect(Collectors.toList());
        stringList.forEach(e -> System.out.println(e));
}
```

#### 查找和匹配
+ allMatch
+ anyMatch
+ noneMatch
+ findFirst
+ findAny



#### 归约
```
//求和,下面的0表示初始值
int sum = numbers.stream().reduce(0, (a, b) -> a + b);

//Lambda
int sum = numbers.stream().reduce(0, Integer::sum);

//无初始值
Optional<Integer> sum = numbers.stream().reduce((a, b) -> (a + b));

//求最大值
Optional<Integer> max = numbers.stream().reduce(Integer::max);
```

map-reduce模式
```
//统计流中有多少个符合的元素
int count = menu.stream()
                .map(d -> 1)        //将每一个元素映射为1然后reduce去累加
                .reduce(0, (a, b) -> a + b);
                
等于与
long count = menu.stream().count();

```


#### 数值流
+ IntStream 
+ DoubleStream 
+ LongStream

将流转换为对应的流对应方法
+ mapToInt
+ mapToLong
+ mapToDouble
```
int calories = menu.stream()
                   .mapToInt(Dish::getCalories)   //返回一个IntStream
                   .sum();                      

                     
//将数值流转换为Stream                     
IntStream intStream = menu.stream().mapToInt(Dish::getCalories);
Stream<Integer> stream = intStream.boxed();

  
// 默认值OptionalInt
OptionalInt maxCalories = menu.stream()
                              .mapToInt(Dish::getCalories)
                              .max();

int max = maxCalories.orElse(-1);
```

数值范围
+ range
+ rangeClosed
```
//表示范围[1, 100]
IntStream evenNumbers = IntStream.rangeClosed(1, 100)
                                 .filter(n -> n % 2 == 0);

System.out.println(evenNumbers.count());
```

#### 流的特点
+ 提取短路
+ 循环合并


### 收集

#### 预定义收集器
Collectors提供的主要功能
+  将流元素归约和汇总为一个值
+ 元素分组
+ 元素分区

常见收集器
+ maxBy/minBy
+ summingInt
+ averagingInt
+ summarizingInt(返回一个统计对象)
+ joining(连接字符串)


```
//求和
int totalCalories = menu.stream().collect(summingInt(Dish::getCalories));
  
//求平均
double avgCalories =
      menu.stream().collect(averagingInt(Dish::getCalories));

 
//统计
IntSummaryStatistics menuStatistics =
        menu.stream().collect(summarizingInt(Dish::getCalories));
//输出的IntSummaryStatistics对象包含{count=9, sum=4300, min=120,average=477.777778, max=800}
        
//连接字符串
String shortMenu = menu.stream().map(Dish::getName).collect(joining());        

```

事实上所有收集器都是可以使用reducing功法方法定义出来的归纳过程的特殊情况而已.  
reducing的三个参数 :   
第一个是归纳操作的起始值  
第二个是函数  
第三个是BinaryOperator  
```
int totalCalories = menu.stream().collect(reducing(
                                   0, Dish::getCalories, (i, j) -> i + j));

```

#### 分组(groupingBy)
```
Map<Dish.Type, List<Dish>> dishesByType =
                      menu.stream().collect(groupingBy(Dish::getType));
  
  
//自定义分组
public enum CaloricLevel { DIET, NORMAL, FAT }

//根据以上枚举分组
Map<CaloricLevel, List<Dish>> dishesByCaloricLevel = menu.stream().collect(
        groupingBy(dish -> {
               if (dish.getCalories() <= 400) return CaloricLevel.DIET;
               else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
               else return CaloricLevel.FAT;
         } ));

```


#### 收集器接口
自定义收集器接口
```java
public interface Collector<T, A, R> {
    /**
    * 创建一个空的累加器实例，供数据收集过程使用
    */
    Supplier<A> supplier();
    
     /**
      *  将元素添加到结果容器
     */
    BiConsumer<A, T> accumulator();
    
    /**
     *   对结果容器应用最终转换
    */
    Function<A, R> finisher();
    
    /**
     *   合并两个结果容器
    */
    BinaryOperator<A> combiner();
    
    /**
    * 定义收集器的行为
    * 
    * Characteristics是一个包含三个项目的枚举。
           UNORDERED——归约结果不受流中项目的遍历和累积顺序的影响。
           
           CONCURRENT——accumulator函数可以从多个线程同时调用，且该收集器可以并行归约流。
                            如果收集器没有标为UNORDERED，那它仅在用于无序数据源时才可以并行归约。
                            
           IDENTITY_FINISH——这表明完成器方法返回的函数是一个恒等函数，可以跳过。
                             这种情况下，累加器对象将会直接用作归约过程的最终结果。
                             这也意味着，将累加器A不加检查地转换为结果R是安全的。

    */
    Set<Characteristics> characteristics();
}
 

/**
* 泛型说明
*  T是流中要收集的项目的泛型。
   A是累加器的类型，累加器是在收集过程中用于累积部分结果的对象。
   R是收集操作得到的对象（通常但并不一定是集合）的类型。
*/


public class ToListCollector<T> implements Collector<T, List<T>, List<T>> {
    
    //创建集合操作的起始点
    @Override
    public Supplier<List<T>> supplier() {
        return ArrayList::new;
    }

    //累积遍历过的项目，原位修改累加器
    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return List::add;
                                                       
    }
    
    //恒等函数
    @Override
    public Function<List<T>, List<T>> finisher() {
        return Function.indentity();
    }
    
    //修改第一个累加器，将其与第二个累加器的内容合并返回修改后的第一个累加器
    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        };
    }
    
    //为收集器添加IDENTITY_FINISH和CONCURRENT标志
    @Override
    public Set<Characteristics> characteristics() {             
        return Collections.unmodifiableSet(EnumSet.of(          
            IDENTITY_FINISH, CONCURRENT));
    }
}

```

进行自定义收集而不去实现Collector
```
List<Dish> dishes = menuStream.collect(
                        ArrayList::new,         //供应源
                        List::add,              //累加器
                        List::addAll);          //组合器
```


###  分支合并框架
+ sequential(顺序流)
+ parallel(并行流)  

并行流背后使用的基础架构是Java 7中引入的分支/合并框架  
分支/合并框架的目的是以递归方式将可以并行的任务拆分成更小的任务，然后将每个子任务的结果合并起来生成整体结果。

```
public abstract  class RecursiveTask<R>{
    protected abstract R compute();
}

//computer的伪代码
if (任务足够小或不可分) {
    顺序计算该任务
} else {
    将任务分成两个子任务
    递归调用本方法，拆分每个子任务，等待所有子任务完成
    合并每个子任务的结果
}

```

```java
                                                                                
    public class ForkJoinSumCalculator extends java.util.concurrent.RecursiveTask<Long> {
        private final long[] numbers;
                                                                     
        private final int start;
                                               
        private final int end;
                                               
        public static final long THRESHOLD = 10_000;              
                                                                  
        public ForkJoinSumCalculator(long[] numbers) {            
            this(numbers, 0, numbers.length);
        }
        
        private ForkJoinSumCalculator(long[] numbers, int start, int end) {
            this.numbers = numbers;
            this.start = start;
            this.end = end;
        }
                                          
                                          
        @Override
        protected Long compute() {
            int length = end - start;
            if (length <= THRESHOLD) {
                 return computeSequentially();
                                                        
             }
             
             //大任务进行任务拆分
             ForkJoinSumCalculator leftTask = new ForkJoinSumCalculator(numbers, start, start + length/2);
             //拆分完后用新线程异步计算
             leftTask.fork();
 
            ForkJoinSumCalculator rightTask = new ForkJoinSumCalculator(numbers, start + length/2, end);
            //拆分后同步递归计算
            Long rightResult = rightTask.compute();
            //阻塞等待结构
            Long leftResult = leftTask.join();
            return leftResult + rightResult;
                                                                   
        }
                                                           
        private long computeSequentially() {
            long sum = 0;
                                                           
            for (int i = start; i < end; i++) {{
                sum += numbers[i];
            }
            return sum;            
        }                          
    }
}

```

```
//调用
public static long forkJoinSum(long n) {
    long[] numbers = LongStream.rangeClosed(1, n).toArray();
    ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
    return new ForkJoinPool().invoke(task);
}
```

#### Spliterator接口
```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
}

```

#### Lambda与匿名内部类的区别
+ 关键字this的含义(Lambda指包含类,匿名指匿名本身的类)
+ 变量隐藏(匿名可以屏蔽包含类的变量,Lambda会编译不通过)

#### Lambda调试
使用peek方法
```
List<Integer> result =
  numbers.stream()
         .peek(x -> System.out.println("from stream: " + x))    //输出来自数据源的当前元素值
         .map(x -> x + 17)
         .peek(x -> System.out.println("after map: " + x))      //输出map的操作结果
         .filter(x -> x % 2 == 0)
         .peek(x -> System.out.println("after filter: " + x))   //输出经过filter后剩下的元素个数
         .limit(3)
         .peek(x -> System.out.println("after limit: " + x))    //输出经过limit之后剩下的元素个数
         .collect(toList());

```