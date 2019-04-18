---
title: Spring Batch 学习（4） <br /> Step 的另一种方式、Listener
date: 2018-11-29 14:44:17
updated: 2018-11-29 14:44:17
categories:
    spring-batch
tags:
    - SpringBatch
---

# 创建步骤的另外一种方式

ItemReader 可以理解为：数据获取。在 Step 中除了可以使用 Tasklet 创建简单的步骤，也可以使用 chunk + itemReader + itemWriter 创建一个复杂的步骤。其中：
> * chunk 表示每几条数据进行一次批量处理
> * itemReader 表示批量获取的数据怎么获取
> * itemWriter 表示 chunk 中的数据进行怎样的输出（到文件、数据库或其他）

<!-- more -->

在使用这种方式创建步骤的时候，需要注意以下几点：
> * chunk 需要指定：输入类型，输出类型、每多少条处理一次，如： <String, String>chunk(10); 表示 ItemReader 的输入类型为 String，ItemWriter 的输出类型为 String，每 10 条处理一次
> * ItemReader 需要指定输入类型，如：ItemReader<String>，指明获取到的数据为 String 类型
> * ItemWriter 需要指定输出类型，如：ItemWriter<String>，指明输出（到文件、数据库或其他）的类型为 String 类型

## 创建一个 String 集合类型的 ItemReader，并进行输出
```java
/**
* @author laiyy
* @date 2018/11/16 9:37
* @description
*/
@Configuration
public class ItemReaderDemo {
   private final JobBuilderFactory jobBuilderFactory;
   private final StepBuilderFactory stepBuilderFactory;
   @Autowired
   public ItemReaderDemo(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory) {
       this.jobBuilderFactory = jobBuilderFactory;
       this.stepBuilderFactory = stepBuilderFactory;
   }
   @Bean
   public Job itemReaderDemoJob() {
       return jobBuilderFactory.get("itemReaderDemoJob")
               .start(itemReaderDemoStep())
               .build();
   }
   @Bean
   public Step itemReaderDemoStep(){
       return stepBuilderFactory.get("itemReaderDemoStep")
                // 指定数据输入为 String，数据输出为 String，每 2 条执行一次
               .<String, String>chunk(2)
               // 指定数据来源 itemReader
               .reader(myStringItemReader())
               // 指定数据数据为打印数据
               .writer( list -> {
                   list.forEach(System.err::println);
               })
               .build();
   }
   /**
     * 声明一个 String 集合，作为数据的输入
     */
   @Bean
   public MyStringItemReader myStringItemReader(){
       List<String> data = Arrays.asList("Cat", "Dog", "Pig", "Duck");
       return new MyStringItemReader(data);
   }
}



public class MyStringItemReader implements ItemReader<String> {
    private Iterator<String> iterator;
    public MyStringItemReader(List<String> data) {
        this.iterator = data.iterator();
    }
    @Override
    public String read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        // 一个数据一个数据的读
        if (iterator.hasNext()) {
            return iterator.next();
        }
        return null;
    }
}
```
---

# Listener

## 几个常用的 Listener

在上一篇博客中提到了 `StepExecutionListener`，这个 Listener 可以在 Step 执行前后获取执行上下文。除此之外还有几个比较常用的 Listener：

> * JobExecutionListener：在 Job 执行前后获取执行上下文
> * ChunkListener：在 chunk 执行前后获取执行上下文
> * ItemReadListener：在 ItemReader 执行前后获取执行上下文
> * ItemWriterListener：在 ItemWriter 执行前后获取执行上下文
> * ItemProcessListener：在 Processor 执行前后获取执行上下文（数据处理器）

在这些 Listener 中，都有 before、after 方法，便于在执行前后获取信息，在实现这些接口，并生成为 `Spring bean` 后，在需要的地方引入即可。

## 比较特殊的几个 Listener 方法：

> * ChunkListsner：afterChunkError，在 chunk 发生错误时调用
> * ItemReaderListener：onReadError，在 itemReader 发生错误时调用
> * ItemWriterListener：onWriteError，在 ItemWriter 发生错误时调用
> * ItemProcessListener：onProcessError，在处理器发生错误时调用

## 另外一种 Listener 实现方式

除了上述实现 xxListener 接口创建 Listener 之外，还有一种更为简单的方式实现 Listener：注解

> * 实现 StepExecutionListener 可以使用： @BeforeStep、@AfterStep
> * 实现 JobExecutionListener 可以使用：@BeforeJob、@AfterJob
> * ...

在这些注解中也有对应 Listener 特殊方法的注解：@AfterChunkError、@OnReadError、@OnWriteError、@OnProcessError

## 代码示例

### 实现接口方式创建 Listener
```java
/**
 * @author laiyy
 * @date 2018/11/15 17:27
 * @description
 *
 * 接口实现监听
 *
 */
public class MyJobListener implements JobExecutionListener {
    @Override
    public void beforeJob(JobExecution jobExecution) {
        // 在 job 执行之前执行
        System.out.println("Job 执行之前，Job 名称：" + jobExecution.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        // 在 job 执行之后执行
        System.out.println("Job 执行之后，Job 名称：" + jobExecution.getJobInstance().getJobName());
    }
}
```

### 注解方式创建 Listener
```java
public class MyChunkListener {

    @BeforeChunk
    public void before(ChunkContext chunkContext){
        System.out.println("Step 执行之前，Step 名称：" + chunkContext.getStepContext().getStepName());
    }

    @AfterChunk
    public void after(ChunkContext chunkContext){
        System.out.println("Step 执行之后，Step 名称：" + chunkContext.getStepContext().getStepName());
    }

}
```

### 使用方式
```java
/**
 * @author laiyy
 * @date 2018/11/15 17:30
 * @description
 *
 * 执行结果
 * Job 执行之前，Job 名称：listenerJob
 * Step 执行之前，Step 名称：listenerStep1
 * （Chunk 设置为2，所以一次拿 2 个数据）
 * Java
 * Python
 * Step 执行之后，Step 名称：listenerStep1
 * Step 执行之前，Step 名称：listenerStep1
 * （两条数据取出来后，step 执行结束，开始获取下一个两条信息）
 * Swift
 * MyBatis
 * Step 执行之后，Step 名称：listenerStep1
 * Step 执行之前，Step 名称：listenerStep1
 * （数据全部取出执行结束）
 * Step 执行之后，Step 名称：listenerStep1
 * Job 执行之后，Job 名称：listenerJob
 *
 */
@Configuration
public class ListenerDemo {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    @Autowired
    public ListenerDemo(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
    }
    @Bean
    public Job listenerJob(){
        return jobBuilderFactory.get("listenerJob")
                .start(listenerStep1())
                // 以创建对象方式引入 JobListener，也可以注入
                .listener(new MyJobListener())
                .build();
    }
    @Bean
    public Step listenerStep1() {
        return stepBuilderFactory.get("listenerStep1")
                // 以 Chunk 方式设置为每读取 2 个数据做一次相应的处理
                // read、process、write；<String, String> 读取为 String，输出为 String
                .<String, String>chunk(2)
                // 容错
                .faultTolerant()
                // 以创建对象方式引入 chunkListener，也可以注入。
                // 设置 Chunk 监听
                .listener(new MyChunkListener())
                // 读取数据
                .reader(itemReader())
                // 输出数据
                .writer(itemWriter())
                .build();

    }
    @Bean
    public ItemWriter<String> itemWriter(){
        return new ItemWriter<String>() {
            @Override
            public void write(List<? extends String> items) throws Exception {
                items.forEach(System.err::println);
            }
        };
    }
    @Bean
    public ItemReader<String> itemReader(){
        return new ListItemReader<>(Arrays.asList("Java", "Python", "Swift", "MyBatis"));
    }
}
```
