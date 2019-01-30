---
title: Spring Batch 学习（2） <br /> 多步骤、步骤嵌套、跳步
date: 2018-11-29 10:52:30
updated: 2018-11-29 10:52:30
categories:
    Java
tags:
    - SpringBatch
---

# SpringBatch 一个任务包含 N 个步骤
在前一篇博客这种，学习到了 SpringBatch 的一个简单的小示例，这个小示例中只包含了一个 Job 任务，这个 Job 也只包含了一个 Step 步骤。
但是有些时候我们需要在一个 Job 中分步骤的处理一些事务，比如：在第一个步骤中需要 *计算所有用户的总资产*，第二个步骤中需要 *计算用户的平均余额*，第三个步骤需要 *筛选男性用户*，第四个步骤.... 这时，就需要在一个 *统计任务* 中分步骤的获取信息。

<!-- more -->

## 代码示例

在上一步（启动一个 Step）的基础上，增加第二个步骤
```java
/**
 * @author laiyy
 * @date 2018/11/15 16:49
 * @description
 */
@Configuration
public class ChildJob2 {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    @Autowired
    public ChildJob2(StepBuilderFactory stepBuilderFactory, JobBuilderFactory jobBuilderFactory) {
        this.stepBuilderFactory = stepBuilderFactory;
        this.jobBuilderFactory = jobBuilderFactory;
    }
    @Bean
    public Job childJobTwo(){
        return jobBuilderFactory.get("childJobTwo")
                // 启动步骤 childJobStep2
                .start(childJobStep2())
                // 在 上一步执行完之后，执行 next 方法，调用下一个步骤
                .next(childJobStep3())
                .build();
    }
    @Bean
    public Step childJobStep3() {
        return stepBuilderFactory.get("childJobStep3")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("childJobStep3");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
    @Bean
    public Step childJobStep2() {
        return stepBuilderFactory.get("childJobStep2")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println("childJobStep2");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
}
```
---

# 一个步骤里面包含多个子步骤
  
在有些业务场景中，可能出现一个任务有多个步骤，而某些步骤又需要包含多个子步骤，如：12306 订票的时候，启动订票任务，订票任务包含：查询票，订票，付款，出票几个步骤，而在查询票的时候，又包含：总余票查询，起始站与终点站间的余票查询，锁定余票等操作。这时就需要在一个步骤里面包含多个子步骤。   


在进行子步骤嵌套的时候需要额外注意 ** 一定要严格审核步骤间的前后关系，防止出现步骤死循环嵌套、前后关系错乱等造成的系统崩溃 **

SpringBatch 中通过使用 Flow 来管理多个子步骤。Flow 可以算是一个步骤，也可以不算一个步骤。 说它算一个步骤，是因为它可以通过 Job 的 start、next 方法，像一个 Step 一个被调用；说它不算一个步骤，是因为 Flow 只起到包含多个子步骤，并按照 Flow 中规定的顺序执行 Step 的作用。

## 代码示例

```java
/**
 * @author laiyy
 * @date 2018/11/15 15:21
 * @description
 */
@Configuration
@EnableBatchProcessing
public class FlowDemo {
    @Autowired
    private JobBuilderFactory jobBuilderFactory;
    @Autowired
    private StepBuilderFactory stepBuilderFactory;
    @Bean
    public Job flowDemoJob(){
        return jobBuilderFactory.get("flowDemoJob")
                // flow 包含了 step1、step3， 则先执行 step1，再执行 step3,
                .start(flowDemoFlow())
                // 再执行 step2
                .next(flowDemoStep2())
                .end()
                .build();
    }
    /**
     * 一个 Flow 含有多个 Step
     * 指明 Flow 对象包含哪些 Step
     */
    @Bean
    public Flow flowDemoFlow(){
        return new FlowBuilder<Flow>("flowDemoFlow")
                .start(flowDemoStep1())
                .next(flowDemoStep3())
                .build();

    }
    @Bean
    public Step flowDemoStep1(){
        return stepBuilderFactory.get("flowDemoStep1").tasklet(new Tasklet() {
            @Override
            public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                System.out.println("flowDemoStep1");
                return RepeatStatus.FINISHED;
            }
        }).build();
    }
    @Bean
    public Step flowDemoStep2(){
        return stepBuilderFactory.get("flowDemoStep2").tasklet(new Tasklet() {
            @Override
            public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                System.out.println("flowDemoStep2");
                return RepeatStatus.FINISHED;
            }
        }).build();
    }
    @Bean
    public Step flowDemoStep3(){
        return stepBuilderFactory.get("flowDemoStep1").tasklet(new Tasklet() {
            @Override
            public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                System.out.println("flowDemoStep3");
                return RepeatStatus.FINISHED;
            }
        }).build();
    }
}
```

---

# 执行跳步

有一些业务场景会有如下需求：前期步骤执行顺序 1->2->3->4->5，但是在运行一段时间之后，需要调整执行顺序为： 1->3->2->5->4，即：当执行完第一个步骤后，执行第三个步骤；然后从第三个步骤开始执行，当第三个步骤执行结束后，执行第二个步骤；当第二个步骤执行结束后，执行第五个步骤；当第五个步骤执行结束后，执行第四个步骤；当第四个步骤执行结束后，任务结束。

这时有两种解决办法，最简单的就是重新给 start、next 赋值。另外一种就是：手动设置步骤执行顺序

## 代码示例
```java
/**
 * @author laiyy
 * @date 2018/11/15 15:06
 * @description
 */
@Configuration
@EnableBatchProcessing
public class JobDemo {
    // 执行成功后会返回的结果
    private static final String COMPLETED = "COMPLETED";
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    @Autowired
    public JobDemo(JobBuilderFactory jobBuilderFactory, StepBuilderFactory stepBuilderFactory) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
    }
    @Bean
    public Job jobDemoJob() {
        return jobBuilderFactory.get("jobDemo")
// 重新给 start、next 赋值
//                .start(stepDemo1())
//                .next(stepDemo2())
//                .next(stepDemo3())
//                .next(stepDemo4())
//                .build();

// 手动设置执行顺序
                // 从 step1, 开始，当 结束 后，到 step2
                .start(stepDemo1()).on(COMPLETED).to(stepDemo2())
                // 从 step2 开始，当结束后，到 step3
                .from(stepDemo2()).on(COMPLETED).to(stepDemo3())
                .from(stepDemo3()).on(COMPLETED).to(stepDemo4())
                // 从 step4 开始，到结束
                .from(stepDemo4()).end().build();
    }
    @Bean
    public Step stepDemo4() {
        return stepBuilderFactory.get("stepDemo4")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println(" stepDemo 4");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
    @Bean
    public Step stepDemo3() {
        return stepBuilderFactory.get("stepDemo3")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println(" stepDemo 3");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
    @Bean
    public Step stepDemo2() {
        return stepBuilderFactory.get("stepDemo2")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println(" stepDemo 2");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
    @Bean
    public Step stepDemo1() {
        return stepBuilderFactory.get("stepDemo1")
                .tasklet(new Tasklet() {
                    @Override
                    public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
                        System.out.println(" stepDemo 1");
                        return RepeatStatus.FINISHED;
                    }
                }).build();
    }
}
```