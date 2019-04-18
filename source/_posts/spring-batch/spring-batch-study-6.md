---
title: Spring Batch 学习（6） <br /> ItemWriter
date: 2018-12-03 09:18:49
updated: 2018-12-03 09:18:49
categories:
    spring-batch
tags:
    - SpringBatch
---

# ItemWriter

`ItemWriter`：批处理之后的数据需要做怎样的写操作（写入文件、数据库、mongodb等）

## 常用的几个 ItemWriter
> * JdbcItemWriter：使用 jdbc 写入数据库
<!-- more -->
> * HibernateWriter：使用 Hibernate 写入数据库
> * JpaItemWriter：使用 JPA 写入数据库
> * JsonFileItemWriter：将数据转为 JSON 写入文本文件
> * FlatFileItemWriter：将数据转为合适格式的字符串，写入文本文件
> * StaxEventItemWriter：将数据通过 oxm 框架的 XStream 写入 xml 文件
> * CompositeItemWriter：写入多个文件
> * ClassifierCompositeItemWriter：文件分类写入
> * ....

## ItemWriter 简单示例（控制台打印数据）

```java
public class MyWriter implements ItemWriter<String> {
    @Override
    public void write(List<? extends String> list) throws Exception {
        // 打印数据长度
        System.out.println(list.size());
        // 遍历打印信息
        list.stream().forEach(System.out::println);
    }
}
```

---

# FlatFileItemWriter

`FlatFileItemWriter`：将数据转为一定格式的字符串，将转换后的字符串写入文本文件中

## 代码示例

使用 Jackson，将数据实体转为 Json 字符串，写入文本（数据来源：
```java
@Bean
public FlatFileItemWriter<City> cityItemWriter(){
    FlatFileItemWriter<City> writer = new FlatFileItemWriter<>();
    String path = "d:\\city.txt";
    writer.setResource(new FileSystemResource(path));
    writer.setLineAggregator(new LineAggregator<City>() {
        @Override
        public String aggregate(City city) {
            try {
                return new ObjectMapper().writeValueAsString(city);
            } catch (JsonProcessingException e) {
                e.printStackTrace();
            }
            return "";
        }
    });
    try {
        writer.afterPropertiesSet();
    } catch (Exception e) {
        e.printStackTrace();
    }
    return writer;
}
```

---

# StaxEventItemWriter

`StaxEventItemWriter` 结合 oxm 框架的 XStream，可以将数据转换位 xml 文件输出

## 代码示例
```java
@Bean
public StaxEventItemWriter<City> xmlItemWriter(){
    StaxEventItemWriter<City> writer = new StaxEventItemWriter<>();

    // xml 处理器
    XStreamMarshaller marshaller = new XStreamMarshaller();

    // 指定根节点
    Map<String, Class> aliases = new HashMap<>(1);
    aliases.put("city", City.class);
    marshaller.setAliases(aliases);

    // setMarshaller：写为 xml，setUnMarshaller：读 xml
    writer.setMarshaller(marshaller);

    // 文件路径
    String path = "d:\\city.xml";
    writer.setResource(new FileSystemResource(path));

    try {
        // 参数校验
        writer.afterPropertiesSet();
    } catch (Exception e) {
        e.printStackTrace();
    }

    return writer;
}
```

---

# CompositeItemWriter、ClassifierCompositeItemWriter

`CompositeItemWriter` 可以将多个文本文件的写入方式结合在一起，实现写入多个文件的操作
`ClassifierCompositeItemWriter` 可以自定义数据分类

## 代码示例

1、写入多个文件
```java
/**
    * 实现数据写入到多个文件
    */
@Bean
public CompositeItemWriter<City> multiFileWriter() throws Exception {
    CompositeItemWriter<City> writer = new CompositeItemWriter<>();
    writer.setDelegates(Arrays.asList(cityItemWriter(), xmlItemWriter()));

    writer.afterPropertiesSet();
    return writer;
}
```

2、文件分类
```java
@Bean
public ClassifierCompositeItemWriter<City> multiFileWriter() {
    ClassifierCompositeItemWriter<City> writer = new ClassifierCompositeItemWriter<>();

    writer.setClassifier(new Classifier<City, ItemWriter<? super City>>() {
        @Override
        public ItemWriter<? super City> classify(City city) {
            // 按照 City 的 id 进行分类
            if (city.getId() % 2 == 0){
                return cityItemWriter();
            } else {
                return xmlItemWriter();
            }
        }
    });
    return writer;
}
```

## 多文件、分类写入注意点

*** 在进行多文件写入时，需要将 ItemWriter 转为 ItemStreamWriter ***：即 在注入时，由注入 ItemWriter<T> 改为 ItemStreamWriter<T>，然后将注入的 ItemWriter 放入 stream 中

示例：
```java
@Autowired
private ItemStreamWriter<City> xmlItemWriter;

@Autowired
private final ItemStreamWriter<City> cityItemWriter;


@Bean
public Step multiFileItemWriterStep(){
    return stepBuilderFactory.get("multiFileItemWriterStep2")
            .<City, City>chunk(10)
            .reader(multiFileReader)
            .writer(multiFileWriter)
            // 将 ItemWriter 放入 stream
            .stream(xmlItemWriter)
            .stream(cityItemWriter)
            .build();
}
```

--- 

# JdbcBatchItemWriter

`JdbcBatchItemWriter` 使用 jdbc 将数据批量写入数据库

## 代码示例
```java
@Bean
public JdbcBatchItemWriter<City> flatFileItemWriter(){
    JdbcBatchItemWriter<City> writer = new JdbcBatchItemWriter<>();
    writer.setDataSource(dataSource);
    writer.setSql("INSERT INTO t_city( id, name, countryCode, district, population) VALUES (:id, :name, :countryCode, :district, :population)");
    // 使用实体属性自动映射到占位符
    writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
    return writer;
}
```

---

# Processor 

`Processor` 是在 ItemReader 到 ItemWriter 之间的一个数据转换操作，如将需要处理的数字转换为字符串等操作

*
ItemProcessor<T, O>，其中： `T` 代表输入类型，即 `ItemReader` 的泛型，`O` 代表输出类型，即 `ItemWriter` 的类型
*
## 代码示例
```java
// 输入类型为 City，输出类型为 City，转换操作为：将城市名称转为大写。
public class NameLowerProcessor implements ItemProcessor<City, City> {
    @Override
    public City process(City city) throws Exception {
        String name = city.getName().toUpperCase();
        city.setName(name);
        return city;
    }
}


// 输入类型为 City，输出类型为 City，转换操作为：如果城市 id 可以被 2 整除，则继续执行，否则跳过
public class IdFilterProcessor implements ItemProcessor<City, City> {
   @Override
   public City process(City city) throws Exception {
       if (city.getId() %2 == 0){
           return city;
       }
       // 如果返回 null，相当于把对象过滤掉
       return null;
   }
}
```

## ItemProcessor 使用

`ItemProcessor` 使用在 Step 域中，尽量书写在 Reader、Writer 中间

`CompositeItemProcessor`：关联多个 Processor

### 代码示例
```java
@Bean
public CompositeItemProcessor<City, City> processListener() {
    CompositeItemProcessor<City, City> processor = new CompositeItemProcessor<>();
    List<ItemProcessor<City, City>> list = new ArrayList<>(2);
    list.add(nameLowerProcessor);
    list.add(idFiltterProcessor);
    // 关联多个 Processor
    processor.setDelegates(list);
    return processor;
}


@Bean
public Step itemProcessorStep() {
    return stepBuilderFactory.get("itemProcessorStep2")
            .<City, City>chunk(10)
            .reader(multiFileReader)
            // 增加 Processor
            .processor(processListener())
            .writer(cityItemWriter)
            .build();
}
```

