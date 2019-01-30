---
title: Spring Batch 学习（5） <br /> ItemReader
date: 2018-11-30 11:42:10
updated: 2018-11-30 11:42:10
categories:
    Java
tags:
    - SpringBatch
---

# ItemReader

`ItemReader` 可以理解为：在批处理中，需要处理的数据。这些数据通常是在 *文本文件*， *xml 文件*， *数据库* 中存储。在进行批处理的时候，需要从文件中获取数据。可以说，ItemReader 是整个批处理的入口。

<!-- more -->

几个常用的 ItemReader

> * ListItemReader：从集合中获取数据
> * FlatFileItemReader：从文本文件中获取数据
> * MultiResourceItemReader：从多个文件中获取数据
> * StaxEventItemReader：从 xml 文件中获取数据
> * AbstractPagingItemReader：从数据库中获取数据：
>> * JdbcPagingItemReader：使用基础的 jdbc 获取数据
>> * HibernatePagingItemReader：使用 Hibernate 获取数据
>> * JpaPagingItemReader：使用 JPA 获取数据

---

# ListItemReader

`ListItemReader`：声明一个集合作为数据的输入，通常用于数据量较小，内存可以处理的批处理操作；数据量过大的话，放入 ListItemReader 中可能造成内存溢出。

## 示例：
```java
@Bean
public ItemReader<String> itemReader(){
    return new ListItemReader<>(Arrays.asList("Java", "Python", "Swift", "MyBatis"));
}
```
---

# FlatFileItemReader

`FlatFileItemReader`：从文本文件中获取数据，这个文本文件可以是 txt、csv 等纯文本文件。
`DelimitedLineTokenizer`：配置数据解析方式。默认以英文逗号为数据分隔符。

## 示例

文本数据(任意增加，但需要保持格式)：
```txt
27,2018-01-28 11:09:26,测试数据1,4500000001,1,1,1,laiyy,1,45000000010215405181645637
28,2018-01-27 01:48:45,测试数据2,4500000001,1,2,1,laiyy,1,45000000010215405181645744
29,2018-01-27 01:48:51,测试数据3,4500000001,1,3,1,laiyy,1,45000000010215405181645843
```

声明一个实体作为每一行文本数据的映射关系：
```java
public class Template {
    private int id;
    private String date;
    private String name;
    private String siteId;
    private int status;
    private int type;
    private int userId;
    private String username;
    private int share;
    private String markId;

    // 省略 get、set
}
```

使用 `FlatFileItemReader` 读取数据
```java
@Bean
@StepScope
public FlatFileItemReader<Template> flatFileReader() {
    FlatFileItemReader<Template> reader = new FlatFileItemReader<>();
    reader.setResource(new ClassPathResource("data.txt"));
    // 跳过第几行
    reader.setLinesToSkip(0);
    // 声明数据解析
    DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();
    // 声明数据分隔符，默认为英文逗号，如果使用其他分隔符需要重新设置
    // tokenizer.setDelimiter(",");
    // 声明每一行分隔符分隔的每个数据代表实体的哪个字段--需要与实体字段名一致
    tokenizer.setNames("id", "date", "name", "siteId", "status", "type", "userId", "username", "share", "markId");
    // 解析后的数据映射为对象
    DefaultLineMapper<Template> mapper = new DefaultLineMapper<>();
    mapper.setLineTokenizer(tokenizer);

    mapper.setFieldSetMapper(new FieldSetMapper<Template>() {
        @Override
        public Template mapFieldSet(FieldSet fieldSet) throws BindException {
            Template template = new Template();
            // 数据映射
            template.setId(fieldSet.readInt("id"));
            template.setDate(fieldSet.readString("date"));
            template.setName(fieldSet.readString("name"));
            template.setSiteId(fieldSet.readString("siteId"));
            template.setStatus(fieldSet.readInt("status"));
            template.setType(fieldSet.readInt("type"));
            template.setUserId(fieldSet.readInt("userId"));
            template.setUsername(fieldSet.readString("username"));
            template.setShare(fieldSet.readInt("share"));
            template.setMarkId(fieldSet.readString("markId"));
            return template;
        }
    });
    // 数据校验
    mapper.afterPropertiesSet();
    // 绑定映射
    reader.setLineMapper(mapper);
    return reader;
}
```

---

# StaxEventItemReader

`StaxEventItemReader`：用于从 xml 文件中读取数据，常常和 spring-oxm 结合使用，极大提高效率

## 代码示例

需要读取的 xml 数据

```xml
<?xml version="1.0" standalone="yes"?>
<RECORDS>
    <RECORD>
        <id>20</id>
        <createDate>2018/5/10 16:23:09</createDate>
        <createUserId>34</createUserId>
        <label>标签1</label>
        <siteId>4500000001</siteId>
        <status>0</status>
        <type>8</type>
        <username>张三</username>
        <seq>20</seq>
        <userId>0</userId>
    </RECORD>
    <RECORD>
        <id>21</id>
        <createDate>2018/5/10 16:24:02</createDate>
        <createUserId>34</createUserId>
        <label>标签2</label>
        <siteId>4500000001</siteId>
        <status>1</status>
        <type>8</type>
        <username>李四</username>
        <seq>22</seq>
        <userId>0</userId>
    </RECORD>
</RECORDS>
```

```java
// 数据实体
public class Label {
    private int id;
    private String label = "";
    private int type;
    private String labelType = "";
    private int userId;
    private int createUserId;
    private String createDate;
    private String username = "";
    private int status = 1;
    private String siteId = "";
    private int seq;

    // 省略 get、set
}


// 数据读取
@Bean
@StepScope
public StaxEventItemReader<Label> xmlFileReader(){
    StaxEventItemReader<Label> reader = new StaxEventItemReader<>();
    // 要读取的文件
    reader.setResource(new ClassPathResource("label.xml"));

    // 指定需要处理的根标签
    reader.setFragmentRootElementName("RECORD");

    // 把读取到的 xml 格式转为 Label
    XStreamMarshaller unmarshaller = new XStreamMarshaller();
    // 设置要读取的 xml 根节点 --- 
    Map<String, Class> map = new HashMap<>(1);
    map.put("RECORD", Label.class);
    unmarshaller.setAliases(map);

    // marshaller ：写 xml
    // unmarshaller： 读 xml
    reader.setUnmarshaller(unmarshaller);
    return reader;
}
```

---

# MultiResourceItemReader

`MultiResourceItemReader`：可以包含多个 ResourceAwareItemReaderItemStream 的子类、实现类 所构建的文件读取 ItemReader，由此可以一次性读取多个文件。

ResourceAwareItemReaderItemStream 的几个常用实现类：
> * FlatFileItemReader：上一例中的文本文件读取
> * JsonItemReader：从 json 文件中获取数据
> * StaxEventItemReader：从 xml 文件中获取数据

## 代码示例

`FlatFileItemWriter` 数据来源：
```
"1","Kabul","AFG","Kabol","1780000"
"2","Qandahar","AFG","Qandahar","237500"
"3","Herat","AFG","Herat","186800"
...
```

```java
/**
 声明一个 txt 文件读取
*/
@Bean
@StepScope
public FlatFileItemReader<City> flatFileReader() {
    FlatFileItemReader<City> reader = new FlatFileItemReader<>();
    // 解析数据
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
    return reader;
}

// 引入 classpath下的所有以 file 开头的 txt 文件
@Value("classpath:file*.txt")
private Resource[] fileResources;

@Bean
@StepScope
public MultiResourceItemReader<City> multiFileReader() {
    MultiResourceItemReader<City> reader = new MultiResourceItemReader<>();
    // 文件读取
    reader.setDelegate(flatFileReader());
    // 将所有文件放入 resources 中，即可实现多文件读取
    reader.setResources(fileResources);
    return reader;
}
```

---

# AbstractPagingItemReader

所有从数据库中分页读取数据的 *** 主抽象类 ***，用于定义分页结构、分页参数等

## JdbcPagingItemReader

以 jdbc 方式分页读取数据（最原生，但是不能自动映射字段，需要自定义）

### 代码示例
```java
/**
* 注解 @StepScope 表示，该数据只在 Step 执行
* JdbcPagingItemReader 从 jdbc 中，使用分页获取数据
*/
@Bean
@StepScope
public JdbcPagingItemReader<User> userItemReader(){
    JdbcPagingItemReader<User> reader = new JdbcPagingItemReader<>();
    reader.setDataSource(dataSource);
    reader.setFetchSize(2);
    // 把读取到的记录转换为 User 对象
    reader.setRowMapper(new RowMapper<User>() {
        @Override
        public User mapRow(ResultSet resultSet, int rowNum) throws SQLException {
            User user = new User();
            user.setId(resultSet.getInt("id"));
            user.setUsername(resultSet.getString("username"));
            user.setPassword(resultSet.getString("password"));
            user.setAge(resultSet.getInt("age"));
            return user;
        }
    });
    // 指定 SQL 语句
    MySqlPagingQueryProvider provider = new MySqlPagingQueryProvider();
    // 查询哪些字段
    provider.setSelectClause("id, username, password, age");
    // 查询哪个表
    provider.setFromClause("from t_user");
    // 根据那个字段进行排序
    Map<String, Order> sortMap = new HashMap<>(2);
    sortMap.put("id", Order.ASCENDING);
    sortMap.put("username", Order.DESCENDING);
    provider.setSortKeys(sortMap);

    reader.setQueryProvider(provider);
    return reader;
}
```

## HibernatePagingItemReader

以 Hibernate 方式分页读取数据（可根据 Hibernate 注解自动映射字段）

### 代码示例

```java
/**
* HibernatePagingItemReader 实现
*/
@Bean
@StepScope
public ItemReader<News> hibernatePagingItemReader() {
HibernatePagingItemReader<News> reader = new HibernatePagingItemReader<>();
    // 每页查询多少条
    reader.setPageSize(500);
    reader.setSessionFactory(sessionFactory);
    HibernateNativeQueryProvider<News> provider = new HibernateNativeQueryProvider<>();
    // 自动映射解析到哪个实体（该实体需要 @Entity 注解）
    provider.setEntityClass(News.class);
    String siteId = parameterMap.get("4500000001").toString();
    int channelId = Integer.valueOf(parameterMap.get("2016").toString());
    StringBuilder sql = new StringBuilder("SELECT * FROM zw_news where site_id =");
    sql.append(siteId);
    if (channelId != 0) {
        sql.append(" and channel_id = ").append(channelId);
    }
    sql.append(" and status = 150 and del_status != 500");
    sql.append(" order by news_weight desc, seq desc, pub_date desc, id desc ");
    provider.setSqlQuery(sql.toString());
    try {
        // 参数校验
        provider.afterPropertiesSet();
    } catch (Exception e) {
        e.printStackTrace();
        throw new RuntimeException("获取 Hibernate 数据失败");
    }
    reader.setQueryProvider(provider);
    // 可有可无
    reader.setQueryName(" News");
    reader.setUseStatelessSession(true);
    return reader;
}
```

