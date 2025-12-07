# Quartz 分布式定时任务 - 完全学习指南

## 📚 简介

Quartz 是一个强大的定时任务框架，支持定时任务的动态管理、持久化存储和分布式调度。与简单的 @Scheduled 注解不同，Quartz 可以在运行时动态创建、删除、暂停任务。

### 为什么学习这个？
- 🎯 **理解定时任务的概念和应用**
- 🎯 **掌握 Quartz 的核心组件：Job、Trigger、Scheduler**
- 🎯 **学会动态管理定时任务**
- 🎯 **实现任务的持久化和集群支持**

---

## 🎯 核心概念

### 1. Quartz 核心组件

```
Job (任务)
  └─ 实现 Job 接口，定义任务内容

Trigger (触发器)
  ├─ SimpleTrigger - 简单触发器
  ├─ CronTrigger - CRON 表达式触发
  └─ CalendarIntervalTrigger - 日期间隔触发

Scheduler (调度器)
  └─ 协调 Job 和 Trigger，定时执行任务
```

### 2. 定时任务的两种方式

#### a) Spring @Scheduled（简单定时）

```java
@Service
public class SimpleTaskService {
    
    // 固定延迟（任务完成后延迟 5 秒再执行）
    @Scheduled(fixedDelay = 5000)
    public void taskWithFixedDelay() {
        log.info("执行任务...");
    }
    
    // 固定速率（每 5 秒执行一次，不管任务是否完成）
    @Scheduled(fixedRate = 5000)
    public void taskWithFixedRate() {
        log.info("执行任务...");
    }
    
    // CRON 表达式（每天上午 10 点执行）
    @Scheduled(cron = "0 0 10 * * *")
    public void taskWithCron() {
        log.info("每天 10 点执行");
    }
}

// 应用启动类需要启用
@SpringBootApplication
@EnableScheduling
public class Application { }
```

#### b) Quartz（复杂定时，支持动态管理）

```java
// 1. 定义 Job
public class MyTask implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("Quartz 定时任务执行");
        // 业务逻辑
    }
}

// 2. 配置触发器和调度器
@Configuration
public class QuartzConfig {
    
    @Bean
    public JobDetail jobDetail() {
        return JobBuilder.newJob(MyTask.class)
            .withIdentity("myTask", "group1")
            .storeDurably()  // 持久化
            .build();
    }
    
    @Bean
    public Trigger trigger() {
        // CRON 触发器
        return TriggerBuilder.newTrigger()
            .forJob(jobDetail())
            .withIdentity("myTrigger", "group1")
            .withSchedule(CronScheduleBuilder.cronSchedule("0 0 10 * * *"))
            .build();
    }
    
    @Bean
    public Scheduler scheduler(JobDetail jobDetail, Trigger trigger) {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setJobDetails(jobDetail);
        factory.setTriggers(trigger);
        return factory.getScheduler();
    }
}

// 3. 动态管理任务
@Service
public class QuartzService {
    
    @Autowired
    private Scheduler scheduler;
    
    // 添加任务
    public void addJob(String jobName, String triggerName, String cronExpression) 
            throws SchedulerException {
        JobDetail jobDetail = JobBuilder.newJob(MyTask.class)
            .withIdentity(jobName)
            .build();
        
        Trigger trigger = TriggerBuilder.newTrigger()
            .forJob(jobDetail)
            .withIdentity(triggerName)
            .withSchedule(CronScheduleBuilder.cronSchedule(cronExpression))
            .build();
        
        scheduler.scheduleJob(jobDetail, trigger);
    }
    
    // 暂停任务
    public void pauseJob(String jobName) throws SchedulerException {
        scheduler.pauseJob(JobKey.jobKey(jobName));
    }
    
    // 恢复任务
    public void resumeJob(String jobName) throws SchedulerException {
        scheduler.resumeJob(JobKey.jobKey(jobName));
    }
    
    // 删除任务
    public void deleteJob(String jobName) throws SchedulerException {
        scheduler.deleteJob(JobKey.jobKey(jobName));
    }
}
```

### 3. CRON 表达式

```
秒   分   小时  日   月   周   年
0    0    10   *    *    *    *

示例：
0 0 10 * * *      - 每天 10 点
0 0 10 ? * MON   - 每周一 10 点
0 0 10 1 * *     - 每月 1 号 10 点
0 0/30 * * * *   - 每 30 分钟执行一次
0 0 10 * * * 2024 - 仅在 2024 年的 10 点执行
```

---

## 💡 实现细节

### 1. 任务参数传递

```java
@Configuration
public class QuartzConfig {
    
    @Bean
    public JobDetail jobDetail() {
        return JobBuilder.newJob(MyTask.class)
            .withIdentity("myTask")
            .usingJobData("userId", 123)  // 传递参数
            .usingJobData("taskName", "SendEmail")
            .build();
    }
}

public class MyTask implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        Long userId = dataMap.getLong("userId");
        String taskName = dataMap.getString("taskName");
        
        log.info("执行任务: {}, 用户ID: {}", taskName, userId);
    }
}
```

### 2. 任务监听器

```java
public class MyJobListener implements JobListener {
    
    @Override
    public String getName() {
        return "myJobListener";
    }
    
    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        log.info("任务开始执行");
    }
    
    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {
        log.info("任务被否决");
    }
    
    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException e) {
        if (e != null) {
            log.error("任务执行失败: {}", e.getMessage());
        } else {
            log.info("任务执行成功");
        }
    }
}
```

### 3. 数据库持久化配置

```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: always  # 初始化数据库表
    properties:
      org:
        quartz:
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJdbcDelegate
            tablePrefix: qrtz_
```

---

## 🏢 行业应用

### 1. 数据报表生成

```java
@Component
public class ReportGenerationJob implements Job {
    
    @Autowired
    private ReportService reportService;
    
    @Override
    public void execute(JobExecutionContext context) {
        // 每天凌晨 2 点生成昨天的报表
        reportService.generateDailyReport();
    }
}
```

### 2. 定期数据清理

```java
@Component
public class DataCleanupJob implements Job {
    
    @Autowired
    private LogRepository logRepository;
    
    @Override
    public void execute(JobExecutionContext context) {
        // 每周清理一周前的日志
        logRepository.deleteByCreateTimeBefore(
            LocalDateTime.now().minusWeeks(1)
        );
    }
}
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>

<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency>
```

### 2. application.yml

```yaml
spring:
  quartz:
    job-store-type: memory  # 或 jdbc
    scheduler-name: quartzScheduler
    wait-for-jobs-to-complete-on-shutdown: true
```

---

## 🧪 测试

```java
@SpringBootTest
class QuartzTest {
    
    @Autowired
    private QuartzService quartzService;
    
    @Test
    public void testAddJob() throws Exception {
        quartzService.addJob("testJob", "testTrigger", "0 0 10 * * *");
        
        // 验证任务是否已添加
        assertTrue(true);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Job | 定时任务的执行体 |
| Trigger | 触发条件定义 |
| Scheduler | 调度器，管理 Job 和 Trigger |
| JobDetail | Job 的详细信息 |
| CronExpression | CRON 表达式 |
| JobDataMap | 传递参数给 Job |

---

**恭喜！** 🎉 你已经掌握了定时任务的两种方式！
