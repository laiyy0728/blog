---
title: Spring Batch 学习（7） <br /> 错误处理
date: 2018-12-03 11:33:31
updated: 2018-12-03 11:33:31
categories:
    Java
tags:
    - SpringBatch
---

# SpringBatch 错误处理

SpringBatch 的错误处理，大致分为：*错误中断，重启后继续执行*，*错误重试*，*错误跳过* 等

<!-- more -->
> * 错误中断，重启后继续执行：在每次 chunk 后在 ExecutionContext 中打入标记，在重启执行该任务时，判断 ExecutionContext 中是否存在标记，如果存在，则从标记位重新读取执行
> * 错误重试：在出现错误时，根据指定的需要*重试*的异常，进行重新读写处理，需要指定：需要重试的异常、重试次数
> * 错误跳过：在出现错误时，根据指定的需要*跳过*的异常，跳过该条数据，需要指定：需要跳过的异常，跳过次数

---

# 错误中断，重启后继续执行

***在 读、处理、写 操作中任何一环出现问题都可以将任务中断***

此项操作，需要 ItemReader、ItemWriter 实现 ItemStreamReader<T>、ItemStreamWriter<T> 接口，在实现类中定义规则

## ItemStreamReader、ItemStreamWriter

实现接口后有以下几个方法需要重写：
> * read()：读取 / 写入 数据的规则
> * open(ExecutionContext executionContext)：在开始读取 / 写入 之前调用，用于第一次执行 或 重启后继续执行时的判断
> * update(ExecutionContext executionContext)：在 chunk 后执行，用于修改数据库中对 ExecutionContext 的记录
> * close()：读取 / 写入 结束后执行

### 代码示例

数据来源(file1.txt)：
```
"1","Kabul","AFG","Kabol","1780000"
"2","Qandahar","AFG","Qandahar","237500"
"3","Herat","AFG","Herat","186800"
"4","Mazar-e-Sharif","AFG","Balkh","127800"
```

```java
public class RestartReader implements ItemStreamReader<City> {

    /**
     * 文件读取
     */
    private FlatFileItemReader<City> reader = new FlatFileItemReader<>();

    /**
     * 当前读到第几行
     */
    private Long curLine = 0L;

    /**
     * 是否重启
     */
    private boolean restart = false;

    /**
     * 执行的上下文
     */
    private ExecutionContext executionContext;

    public RestartReader() {
        reader.setResource(new ClassPathResource("file1.txt"));

        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();
        tokenizer.setNames("id", "name", "countryCode", "district", "population");
        // 解析后的数据映射为对象
        DefaultLineMapper<City> mapper = new DefaultLineMapper<>();
        mapper.setLineTokenizer(tokenizer);

        mapper.setFieldSetMapper(new FieldSetMapper<City>() {
            @Override
            public City mapFieldSet(FieldSet fieldSet) throws BindException {
                City city = new City();
                city.setCountryCode(fieldSet.readString("countryCode"));
                city.setDistrict(fieldSet.readString("district"));
                city.setId(fieldSet.readInt("id"));
                city.setName(fieldSet.readString("name"));
                city.setPopulation(fieldSet.readLong("population"));
                return city;
            }
        });
        // 数据校验
        mapper.afterPropertiesSet();
        // 绑定映射
        reader.setLineMapper(mapper);
    }

    @Override
    public City read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        City city = null;
        // 每次读取数据，当前行 +1
        this.curLine++;

        if (restart) {
            // 如果是重启（出现错误之后），则从 chunk 记录行开始读取
            reader.setLinesToSkip(this.curLine.intValue() - 1);
            // 将重启值置为 false，否则将会重复读取
            restart = false;
            System.out.println("Start reading from line: " + this.curLine);
        }
        reader.open(this.executionContext);

        city = reader.read();
//        模拟出现错误：读到第 100 行数据时出错
//        if (city != null && this.curLine == 100) {
//            throw new RuntimeException("Something Wrong!");
//        }
        return city;
    }

    /**
     * 在开始读取之前调用
     */
    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        // 获取当前执行上下文
        this.executionContext = executionContext;
        if (executionContext.containsKey("curLine")) {
            // 如果执行上下文存在 cutLine，则证明执行为 重启后执行
            this.curLine = executionContext.getLong("curLine");
            // 将重启值置为 true
            this.restart = true;
        } else {
            // 第一次执行，向执行上下文中打入 curLine 记录（会记录进数据库）
            this.curLine = 0L;
            executionContext.put("curLine", 0L);
            System.out.println("Start reading from line: " + (this.curLine + 1));
        }
    }

    /**
     * 在每次读取 chunk 条数据后调用
     */
    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        // 每次 chunk 后，重新打入 curLine 为当前行（会记录进数据库）
        executionContext.put("curLine", this.curLine);
        System.out.println("Reading line: " + (this.curLine + 1));

    }

    @Override
    public void close() throws ItemStreamException {

    }
}
```

---

# 错误重试

*** 在 读、处理、写 操作中任何一环出现问题都可以重新执行出现错误的 chunk ***

## 代码示例

### 模拟在 Processor 中出现错误
```java
@Component
public class RetryProcessor implements ItemProcessor<String, String> {

    private int attemptCount = 0;

    @Override
    public String process(String item) throws Exception {
        System.out.println("processing item :" + item);
        // 模拟错误：如果需要处理的数据为字符串 26，判断重试次数，如果重试次数大于等于 3 次，则数据处理成功，否则抛出异常，处理处理失败
        if ("26".equalsIgnoreCase(item)) {
            attemptCount++;
            if (attemptCount >= 3) {
                System.out.println("Retried " + attemptCount + "times success");
                return String.valueOf(Integer.valueOf(item) * -1);
            }
            System.out.println("Processed the " + attemptCount + " times fail");
            throw new RuntimeException("Process failed. Attempt: " + attemptCount);
        }
        return String.valueOf(Integer.valueOf(item) * -1);
    }
}
```

### 在 Step 中进行错误重试操作

```java

@Bean
@StepScope
public ListItemReader<String> reader(){
    List<String> items = new ArrayList<>();
    for (int index = 0; index< 60; index++){
        items.add(""+index);
    }
    return new ListItemReader<>(items);
}

@Bean
public Step retryDemoStep(){
    return stepBuilderFactory.get("retryDemoStep")
            .<String, String>chunk(10)
            .reader(reader())
            .processor(retryItemProcessor)
            .writer(retryItemWriter)
            // 容错
            .faultTolerant()
            // 发生哪个异常时进行重试
            .retry(RuntimeException.class)
            // 重试几次
            .retryLimit(10)
            .build();
}
```

在此时，运行程序后，会发现控制台打印 0-20，30-60 都正常，但是在带引 20 - 30 的数据时，由于在 26 处出现了错误，会多次打印 20-25，和错误信息："Processed the " + attemptCount + " times fail"

由此可证明错误重试 成功

---

# 错误跳过

*** 在 读、处理、写 操作中任何一环出现问题都可以重新执行出现错误的 chunk ***

## 代码示例

出现的错误还是以上例中的错误为本例错误

```java
@Bean
public Step skipDemoStep(){
    return stepBuilderFactory.get("skipDemoStep")
            .<String, String>chunk(10)
            .reader(reader())
            .processor(retryItemProcessor)
            .writer(retryItemWriter)
            // 容错
            .faultTolerant()
            // 跳过
            .skip(RuntimeException.class)
            // 跳过次数
            .skipLimit(10)
            .build();
}
```

此时运行代码，可以发现，当 26 错误错误时，processor 自动略过，在 ItemWriter 中并没有打印信息，控制台打印信息为： ... 23  24  25  27  29 ...

由此可看出 26 被成功跳过，则错误跳过成功

---

# 错误处理监听器

错误处理监听器：可以在执行批处理时，在出现错误的地方通过监听器，监听错误信息，如：read error、write error、processor error

## 常见的错误处理监听器

> * SkipListener：错误跳过监听
> * RetryListener：错误重试监听，该 listener 本身不提供操作，由以下几个子 Listener 提供操作
>> * RetryProcessListener：processor error 消息监听
>> * RetryWriteListener：write error 消息监听
>> * RetryReadListener：read error 消息监听

## 代码示例

出现错误的方式还是以上例中的 字符串 26 错误为例

** 以 SkipListener 为例 **
```java
@Component
public class MySkipListener implements SkipListener<String, String> {

    /**
     * 读取跳过
     * @param throwable 发生的异常
     */
    @Override
    public void onSkipInRead(Throwable throwable) {

    }

    /**
     * 写入错误
     * @param s 写入的数据
     * @param throwable 发生的异常
     */
    @Override
    public void onSkipInWrite(String s, Throwable throwable) {

    }

    /**
     * 在处理流程中出现的异常
     * @param s 出现异常的数据
     * @param throwable 出现的异常
     */
    @Override
    public void onSkipInProcess(String s, Throwable throwable) {
        System.out.println(s + " ----> " + throwable.getLocalizedMessage());
    }
}
```

Listener 使用：
```java
@Bean
public Step skipListenerStep(){
    return stepBuilderFactory.get("skipListenerStep")
            .<String, String>chunk(10)
            .reader(reader())
            .writer(skipItemWriter)
            .processor(skipItemProcessor)
            .faultTolerant()
            .skip(RuntimeException.class)
            // 指定错误处理 Listener
            .listener(skipListener)
            .skipLimit(10)
            .build();
}
```





