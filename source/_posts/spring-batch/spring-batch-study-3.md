---
title: Spring Batch 学习（3） <br /> 决策器、JobParameters、Step 监听器
date: 2018-11-29 13:37:49
updated: 2018-11-29 13:37:49
categories:
    spring-batch
tags:
    - SpringBatch
---

# 决策器

一些业务比较复杂的批处理操作中，可能会存在如下的需求：如在 *微博抽奖* 中，进行批量处理挑选中间人的时候，需要根据 *活跃度*，*发帖量*，*粉丝数* 的不同，进行不同筛选操作。这时就需要一个 *** 决策器 ***，决策器的作用：根据不同的条件，返回不同的状态码（自定义状态码），根据状态码的不同，选择不同的步骤进行批量处理操作。

<!-- more -->

## 代码示例

### 自定义一个决策器

自定义一个决策器，当输入值为 *奇数* 的时候，返回 "odd"，当输入值为 *偶数* 的时候，返回 "even"

```java
/**
* @author laiyy
* @date 2018/11/15 16:23
* @description
*
* 自实现的决策器
*/
public class MyDecider implements JobExecutionDecider {

   // 总处理条数
   private int count;

   @Override
   public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
       // 每次进入决策器，则处理条数加 1
       count++;
       if (count % 2 == 0) {
           // 如果总条数能被 2 整除，返回 even
           return new FlowExecutionStatus("even");
       }else {
           // 否则返回 odd
           return new FlowExecutionStatus("odd");
       }
   }
}
```

### 使用自定义的决策器，进行步骤选择
```java
/**
 * @author laiyy
 * @date 2018/11/15 16:10
 * @description
 */
@Configuration
@EnableBatchProcessing
public class DeciderDemo {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    @Autowired
    public DeciderDemo(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
    }
    /**
     * 创建任务2
     */
    @Bean
    public Job deciderDemoJob(){
        return jobBuilderFactory.get("deciderDemoJob")
                .start(deciderDemoStep1())
                .next(myDecider())
                // 如果决策器返回的是 even，进入 step2
                .from(myDecider()).on("even").to(deciderDemoStep2())
                // 如果决策器返回的是 odd，进入 step3
                .from(myDecider()).on("odd").to(deciderDemoStep3())
                // 由于先执行的是 step3，那么无论决策器返回值是什么都重新进入决策器，即：进入 next(myDecider())，这时会进入 step2 执行。
                // 如果不加这句，则只会执行 step3，不会执行 step2
                .from(deciderDemoStep3()).on("*").to(myDecider())
                .end().build();
    }
    /**
     * 决策器
     */
    @Bean
    public JobExecutionDecider myDecider(){
        return new MyDecider();
    }
    @Bean
    public Step deciderDemoStep3() {
        return stepBuilderFactory.get("deciderDemoStep3")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("odd");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
    @Bean
    public Step deciderDemoStep2() {
        return stepBuilderFactory.get("deciderDemoStep2")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("even");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
    @Bean
    public Step deciderDemoStep1() {
        return stepBuilderFactory.get("deciderDemoStep1")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("deciderDemoStep1");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
}
```

---

# JobParamters 
`JobParameters`，就是Job运行时的参数。它在bath中有两个作用：一是标示不同的 `JobInstance` 实例，二是作为 job 中用到的信息，以参数的形式传给job。

通常每个 Job 都会有不通的启动方式，或者启动参数等信息，所以一般来说，在一个 Job 中，会公用一个 `JobParamters`。

一般来说，不会在 Job 中使用 `JobParamters`，大部分情况下，是在 Job 的执行步骤中使用，即在 Step 中使用 `JobParamters`。在不同的 Step 中获取需要的 `JobParamters`，更加便于 Step 的执行（比如需要判断 `JobParamerts` 的内容是否是正在执行的 Step 所需要的，如果需要就获取执行，如果不需要就略过）。

## StepExecutionListener

在 Step 中获取 `JobParameters`，通常使用 `StepExecutionListener` 监听器。这个监听器的作用是：在某个 Step 中传入监听器，这个监听器就可以获取到这个 Step 的所有上下文信息。在监听器的是实现方法中，进行上下文信息的获取、`JobParameters` 的获取、Step 执行前后的上下文处理等操作。

StepExecutionListener 需要实现 2 个方法：
> * beforeStep(StepExecution stepExecution)：在 Step 执行前获取 Step 的上下文信息
> * afterStep(StepExecution stepExecution)：在 Step 执行后获取 Step 的上下文信息

### 代码示例
```java
/**
 * @author laiyy
 * @date 2018/11/15 17:27
 * @description
 *
 * 接口实现监听
 *
 */
public class MyStepListener implements StepExecutionListener {

    /**
    * 存储 Job 的参数
    */
   private Map<String, JobParameter> parameterMap;

    @Override
    public void beforeStep(SteoExecution stepExecution) {
        // 在 Step 执行之前执行
        System.out.println(" 执行之前，Step 名称：" + stepExecution.getStepName());
        // 获取 JobParameters
        parameterMap = stepExecution.getJobParameters().getParameters();
    }

    @Override
    public void afterStep(StepExecution stepExecution) {
        // 在 Step 执行之后执行
        System.out.println(" 执行之后，Step 名称：" + stepExecution.getStepName());
    }
}
```

## 在 Job 运行期间获取 JobParameters

在 Job 运行期间获取 JobParameters，有两种方法：
> * Map<String, JobParameter> 声明为公开静态变量，在 Listener 中使用
> * 声明 Job 的类实现 StepExecutionListener 接口，直接在本类中使用私有变量调用

### 代码示例（以第二种为例）
```java
/**
 * @author laiyy
 * @date 2018/11/16 9:22
 * @description
 */
@Configuration
public class ParametersDemo implements StepExecutionListener {

    private final JobBuilderFactory jobBuilderFactory;

    private final StepBuilderFactory stepBuilderFactory;

    /**
     * 存储 Job 的参数
     */
    private Map<String, JobParameter> parameterMap;

    @Autowired
    public ParametersDemo(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
    }

    @Bean
    public Job parameterJob(){
        return jobBuilderFactory.get("parameterJob")
                .start(parameterStep())
                .build();
    }

    /**
     * Job 执行的是 Step，所以 Job 的参数是在 Step 中使用
     * 所以需要给 Step 传递参数即可
     * 可以使用监听的方式传递数据：即使用 Step 级别的监听在 Step 执行之前传递数据
     */
    @Bean
    public Step parameterStep(){
        return stepBuilderFactory.get("parameterStep")
                // 使用 this 关键字，即使用 ParametersDemo 的实例作为 Listener
                .listener(this)
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("【parameterStep】接收到的参数：" + parameterMap.get("info"));
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }


    @Override
    public void beforeStep(StepExecution stepExecution) {
        System.out.println("在 Step 执行之前传入参数");
        // stepExecution.getJobParameters().getParameters() 是在项目执行时传入的参数，即：项目运行参数
        parameterMap = stepExecution.getJobParameters().getParameters();
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        System.out.println("在 Step 执行之后处理结果");
        return null;
    }
}
```
