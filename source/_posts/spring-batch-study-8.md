---
title: Spring Batch 学习（8） <br />  JobLauncher、JobOperator、事务处理
date: 2018-12-24 14:15:11
updated: 2018-12-24 14:15:11
categories:
    Java
tags:
    - SpringBatch
---

现在是所有实例，都是在 SpringBoot 中，在启动项目的同时，进行任务、步骤的构建，任务的启动。但是有时需要在一个 Controoler、或者一个 Scheduler 中进行任务的调度，这时使用现在的方式就不合适了。


<!-- more -->

# 在 SpringBoot 中禁用 batch 自启动

在 application.yml 文件中，将 `spring.batch.job.enabked` 设置为 false，即可禁用自启动 SpringBatch，但是任务仍然会构造，只是不会自动执行。

---

# JobLauncher

`JobLaunch` 是手动启动 SpringBatch 任务的启动类。需要参数：任务实例、任务执行中的参数`(JobParameters)`

## 实现方式：

在需要启动任务的地方，如：Controoler 中注入 JobLauncher，构建 JobParameters，启动指定的任务

```java
@RestController
public class JobController {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job job;

    @GetMapping(value = "/job/{msg}")
    public String jobRun1(@PathVariable String msg){
        // 构建 JobParameters
        JobParameters jobParameters = new JobParametersBuilder()
                .addString("msg", msg)
                .toJobParameters();

        // 启动任务，并把参数传给任务
        try {
            jobLauncher.run(jobLauncherDemoJob, jobParameters);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "job success";
    }

}
```

需要注意的点：

> 1. 构建的 JobParameters 会在调用 jobLauncher.run 时，实例化到数据库中，如果执行过一次之后，再次执行需要保证参数不一样，否则会任务该任务已经执行过，不能再次执行。
> 2. 如果有多个 Job，需要使用 @Qualifier 指定要注入的 Job
> 3. 这种方式启动任务，任务的运行是 **同步** 的，不是异步的。

## 异步 JobLauncher：

异步的 JobLauncher 需要手动设置线程池、任务执行的 repository 持久化
```java
@Autowired
private JobRepository jobRepository;

@Bean
public JobLauncher jobLauncher() {
    SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
    // jobRepository 需要注入
    jobLauncher.setJobRepository(jobRepository);
    // 使用 Spring 线程池，可以自定义
    jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor());
    return jobLauncher;
}
```

自定义线程池：
```java
@Bean
public Executor getExecutor(){
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);
    executor.setMaxPoolSize(100);
    executor.setQueueCapacity(150);
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(60);
    executor.setThreadNamePrefix("batch-");
    executor.setRejectedExecutionHandler(new DiscardOldestPolicy());
    executor.initialize();
    return executor;
}
```

---

# JobOperator

JobOperator 可以任务是对 JobLauncher 的再次封装，所有在 JobOperator 中需要注入 JobLauncher

## 构建 JobOperator

```java
 @Autowired
private JobLauncher jobLauncher;

@Autowired
private JobRepository jobRepository;

@Autowired
private JobExplorer jobExplorer;

@Autowired
private JobRegistry jobRegistry;

@Bean
public JobOperator jobOperator(){
    SimpleJobOperator jobOperator = new SimpleJobOperator();
    // 注入 JobLauncher
    jobOperator.setJobLauncher(jobLauncher);
    // 设置参数转换器
    jobOperator.setJobParametersConverter(new DefaultJobParametersConverter());
    // 注入 jobRepository 持久化
    jobOperator.setJobRepository(jobRepository);
    // 注入 任务注册器
    jobOperator.setJobRegistry(jobRegistry);
    // 注入任务探测器
    jobOperator.setJobExplorer(jobExplorer);
    return jobOperator;
}
```

## 构建 jobRegistry Processor

```java
public class JobOperatorConfig implements ApplicationContextAware {
    private ApplicationContext applicationContext;

    @Bean
    public JobRegistryBeanPostProcessor jobRegistry(){
        JobRegistryBeanPostProcessor processor = new JobRegistryBeanPostProcessor();
        processor.setJobRegistry(jobRegistry);
        processor.setBeanFactory(applicationContext.getAutowireCapableBeanFactory());
        try {
            processor.afterPropertiesSet();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return processor;

    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

## 执行任务
```java
try {
    jobOperator.start("jobName", "msg="+msg);
} catch (NoSuchJobException | JobInstanceAlreadyExistsException | JobParametersInvalidException e) {
    e.printStackTrace();
}
```

## JobLauncher 与 JobOperator 的比较

> 执行方法： JobLauncher 使用 run 方法执行，JobOperator 使用 start 方法执行
> 任务注入： JobLauncher 使用 Job 实例注入，JobOperator 使用任务名称注入
> 参数传递： JobLauncher 使用 JobParameters 传递，JobOperator 使用字符串传递、且参数传递方式为 key1=value1&key2=value2...
> 参数转换： JobLauncher 不需要转换，JobOperator 需要设置参数转换器才能转换为 JobParameters

--- 

# 事务处理 -- 重中之重

在一些批处理任务重，可能会需要用到数据库。如果用到了数据库的读、写操作，就一定会牵扯到事务问题。

## 在 SpringBoot 中的事务处理

在构建 Job 时，需要显式注入需要使用的事务处理器，并传入 Step。

```java
@Bean
public Job jobDemo(PlatformTransactionManager transactionManager){
    return jobBuilderFactory.get("jobDemo")
            .start(steoDemo(transactionManager))
            .build();
}

@Bean
public Step steoDemo(PlatformTransactionManager transactionManager) {
    return stepBuilderFactory.get("steoDemo")
            .transactionManager(transactionManager)
            .tasklet(new Tasklet() {
                @Override
                public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                    System.out.println("childJobStep1");
                    return RepeatStatus.FINISHED;
                }
            }).build();
}

```

这种方式是使用 SpringBatch 的自带事务处理器进行事务处理，但是在一些集成了 Hibernate、MyBatis 的系统中，需要用到 Hibernate、MyBatis 的事务处理器。如果此时，使用的是 SpringBoot 项目，且事务处理器没有自定义的话，是没有用问题的。
如果是使用的 SpringMVC 进行任务调用，且自定义了事务处理器，这时可能出现问题。

## 在 SpringMVC 自定义事务处理器的问题

自定义事务处理器：
```xml
<!-- 定义事务 -->
<bean id="txAdvice" class="org.springframework.orm.hibernate5.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory" />
</bean>

<tx:annotation-driven transaction-manager="txAdvice"/>

<!-- 拦截器方式配置事物  -->
<tx:advice id="transactionAdvice" transaction-manager="txAdvice">
    <tx:attributes>
        <tx:method name="update*" propagation="REQUIRED" />
        <tx:method name="add*" propagation="REQUIRED" />
        <tx:method name="save*" propagation="REQUIRED" />
        <tx:method name="edit*" propagation="REQUIRED" />
        <tx:method name="delete*" propagation="REQUIRED" />
        <tx:method name="del*" propagation="REQUIRED"/>
        <tx:method name="remove*" propagation="REQUIRED" />
        <tx:method name="repair*" propagation="REQUIRED"/>
        <tx:method name="*"  read-only="true" />
    </tx:attributes>
</tx:advice>
```

在批处理任务中，没有显示注入事务处理器，此时在执行批处理时，会有如下问题：

### 控制台错误打印

```console
2018-12-24 15:05:11.809 WARN [NettyClientWorkerThread_1][NettyRemotingAbstract.java:258] - RemotingCommand [code=17, language=JAVA, version=252, opaque=10, flag(B)=1, remark=No topic route info in name server for the topic: TBW102
See http://rocketmq.apache.org/docs/faq/ for further details., extFields=null, serializeTypeCurrentRPC=JSON]
2018-12-24 15:05:11.811 ERROR [SimpleAsyncTaskExecutor-2][AbstractStep.java:229] - Encountered an error executing step firstStepOfFindSite in job batchGenerateChannelJob
org.springframework.transaction.CannotCreateTransactionException: Could not open Hibernate Session for transaction; nested exception is java.lang.IllegalStateException: Already value [org.springframework.jdbc.datasource.ConnectionHolder@76ce6385] for key [{
	CreateTime:"2018-12-24 15:01:21",
	ActiveCount:2,
	PoolingCount:18,
	CreateCount:20,
	DestroyCount:0,
	CloseCount:30,
	ConnectCount:32,
	Connections:[
		{ID:428371115, ConnectTime:"2018-12-24 15:01:21", UseCount:0, LastActiveTime:"2018-12-24 15:01:21"},
		...
		{ID:1832136599, ConnectTime:"2018-12-24 15:01:22", UseCount:0, LastActiveTime:"2018-12-24 15:01:22"}
	]
}

[
	{
	ID:428371115, 
	poolStatements:[
		]
	},
	...
	{
	ID:1832136599, 
	poolStatements:[
		]
	}
]] bound to thread [SimpleAsyncTaskExecutor-2]
	at org.springframework.orm.hibernate5.HibernateTransactionManager.doBegin(HibernateTransactionManager.java:542) ~[spring-orm-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:377) ~[spring-tx-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.transaction.interceptor.TransactionAspectSupport.createTransactionIfNecessary(TransactionAspectSupport.java:461) ~[spring-tx-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:277) ~[spring-tx-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96) ~[spring-tx-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) ~[spring-aop-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at com.alibaba.druid.support.spring.stat.DruidStatInterceptor.invoke(DruidStatInterceptor.java:72) ~[druid-1.1.6.jar:1.1.6]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) ~[spring-aop-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:213) ~[spring-aop-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at com.sun.proxy.$Proxy79.getOne(Unknown Source) ~[?:?]
	at com.dahe.wang.batch.BatchGenerateConfig$2.execute(BatchGenerateConfig.java:195) ~[classes/:?]
	at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:406) ~[spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:330) ~[spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:133) ~[spring-tx-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.batch.core.step.tasklet.TaskletStep$2.doInChunkContext(TaskletStep.java:272) ~[spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.scope.context.StepContextRepeatCallback.doInIteration(StepContextRepeatCallback.java:81) ~[spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.repeat.support.RepeatTemplate.getNextResult(RepeatTemplate.java:374) ~[spring-batch-infrastructure-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.repeat.support.RepeatTemplate.executeInternal(RepeatTemplate.java:215) ~[spring-batch-infrastructure-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.repeat.support.RepeatTemplate.iterate(RepeatTemplate.java:144) ~[spring-batch-infrastructure-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.step.tasklet.TaskletStep.doExecute(TaskletStep.java:257) ~[spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.step.AbstractStep.execute(AbstractStep.java:200) [spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.job.SimpleStepHandler.handleStep(SimpleStepHandler.java:148) [spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.job.AbstractJob.handleStep(AbstractJob.java:392) [spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.job.SimpleJob.doExecute(SimpleJob.java:135) [spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.job.AbstractJob.execute(AbstractJob.java:306) [spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at org.springframework.batch.core.launch.support.SimpleJobLauncher$1.run(SimpleJobLauncher.java:135) [spring-batch-core-3.0.9.RELEASE.jar:3.0.9.RELEASE]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_171]
Caused by: java.lang.IllegalStateException: Already value [org.springframework.jdbc.datasource.ConnectionHolder@76ce6385] for key [{
	CreateTime:"2018-12-24 15:01:21",
	ActiveCount:2,
	PoolingCount:18,
	CreateCount:20,
	DestroyCount:0,
	CloseCount:30,
	ConnectCount:32,
	Connections:[
		{ID:428371115, ConnectTime:"2018-12-24 15:01:21", UseCount:0, LastActiveTime:"2018-12-24 15:01:21"},
		...
		{ID:1832136599, ConnectTime:"2018-12-24 15:01:22", UseCount:0, LastActiveTime:"2018-12-24 15:01:22"}
	]
}

[
	{
	ID:428371115, 
	poolStatements:[
		]
	},
	...
	{
	ID:1832136599, 
	poolStatements:[
		]
	}
]] bound to thread [SimpleAsyncTaskExecutor-2]
	at org.springframework.transaction.support.TransactionSynchronizationManager.bindResource(TransactionSynchronizationManager.java:190) ~[spring-tx-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	at org.springframework.orm.hibernate5.HibernateTransactionManager.doBegin(HibernateTransactionManager.java:516) ~[spring-orm-4.3.18.RELEASE.jar:4.3.18.RELEASE]
	... 26 more
```

### 使用 debeug，在第一个进行数据库操作的位置打断点，可以看到 debugger 下的如下错误：
![debugger](/images/spring-batch/transaction_manager.png)

debugger 显示：不能创建事务。

错误原因： SpringBatch 默认使用 jdbc 的事务管理器，而没有使用自定义的 Hibernate 事务管理器。这时，在整个项目中，同时存在 Hibernate 和 Jdbc 的事务管理器，SpringBatch 无法判断使用哪个事务管理器，导致事务嵌套，从而抛出异常。

SpringBatch 会使用名为 `transactionManager` 的事务管理器，而在本例中， xml 设置的事务管理器，名为 `txAdvice`，从而导致出现多事务。

解决办法： 将声明式事务 `txAdvice` 名称修改为 `transactionManager` 即可。


*** 在 Spring + SpringBatch 项目中，要严防出现事务嵌套！！！ ***