## CompletableFuture初探

Java 在 1.8 版本提供了 CompletableFuture 来支持**异步编程**,默认情况下 CompletableFuture 会使用公共的 ForkJoinPool 线程池，这个线程池默认创建的线程数是 CPU 的核数,但强烈建议你要根据不同的业务类型创建不同的线程池，以避免互相干扰。下面是一些简单的运用

```java
public class CompletableFutureTest {
    //单独定义一个线程池,为了简洁使用使用默认提供的固定线程池(生产环境不要这么用)
    private static ExecutorService executorService  = Executors.newFixedThreadPool(4);

    public static void main(String[] args) throws Exception {
        //helloWorld();

        //serialFuture();

        //combineFuture();

        exceptionHandle();

        executorService.shutdown();
    }

    /**
     * 使用CompletableFuture创建一个异步函数
     */
    private static void helloWorld() throws ExecutionException, InterruptedException {
        //创建一个无返回值的异步函数
        CompletableFuture future = CompletableFuture.runAsync(()->
            System.out.println("HelloWorld")
        ,executorService);

        //创建一个有返回值的异步函数,类似于Future
        CompletableFuture supplyFuture = CompletableFuture.supplyAsync(()-> "Happy New Year",executorService);
        System.out.println(supplyFuture.join());
    }


    /**
     *  异步后的串行
     */
    private static void serialFuture() throws ExecutionException, InterruptedException {
        CompletableFuture future = CompletableFuture
            	//1  第一个异步返回一个Hello字符串  异步
                .supplyAsync(()->"Hello",executorService)                     
            	//2  依赖于1拼凑成"Hello World"      同步
                .thenApply(s -> s + " World")         
                //3  将2的结果全转换为大写            同步
                .thenApply(String::toUpperCase)                               
                //4  将3的值输出
                .thenAccept((val)->System.out.println(val));                  

        future.join();
    }

    /**
     *  汇总的异步,如f3需等待f1,f2之后才执行
     */
    private static void combineFuture(){
        //模拟一个无返回的异步线程
        CompletableFuture f1 = CompletableFuture.runAsync(()-> System.out.println(1));
        //模拟一个需用到其返回值的异步线程
        CompletableFuture f2 = CompletableFuture.supplyAsync(()-> "Hello");

        //先执行f1,再执行f2,之后f3获取到f2的结果再执行最后的汇总结果
        CompletableFuture f3 = f1.thenCombine(f2,(empty,val)->{
           String word = val +  " World";
           return word;
        });
        System.out.println(f3.join());
    }

    /**
     * 异常处理
     * whenComplete() 和 handle() 的区别在于 whenComplete() 不支持返回结果，而 handle() 是支持返回结果的。
     */
    private static void exceptionHandle(){
        CompletableFuture future = CompletableFuture
                .supplyAsync(()-> 7/0)
                .thenApply(i -> i * 10)
                .exceptionally(e -> 0) //异常发生时的返回值
                .whenComplete((i,e)->{
                    System.out.println("返回值为" + i);
                    System.out.println("异常为" + e.getMessage());
                });
        System.out.println(future.join());
    }

}
```

