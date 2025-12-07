# 简单定时任务 - 完全学习指南

## 📚 简介

Spring Boot 的 @Scheduled 注解提供了简单的定时任务支持，适合轻量级的定时需求。无需额外配置，即可快速实现定时执行。

### 核心特点
- 🎯 **简单易用：仅需一个注解**
- 🎯 **零配置：自动支持，无需显式配置**
- 🎯 **灵活调度：支持固定延迟、固定速率、CRON 表达式**

---

## 🎯 使用方法

### 1. 启用定时任务

```java
@SpringBootApplication
@EnableScheduling  // 启用定时任务
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2. 定义定时任务

```java
@Component
public class ScheduledTasks {
    
    // 固定延迟（任务完成后延迟 5 秒再执行）
    @Scheduled(fixedDelay = 5000)
    public void taskWithFixedDelay() {
        System.out.println("固定延迟任务: " + new Date());
    }
    
    // 固定速率（每 5 秒执行一次，不管任务是否完成）
    @Scheduled(fixedRate = 5000)
    public void taskWithFixedRate() {
        System.out.println("固定速率任务: " + new Date());
    }
    
    // 初始延迟 + 固定速率
    @Scheduled(initialDelay = 2000, fixedRate = 5000)
    public void taskWithInitialDelay() {
        System.out.println("延迟启动的定时任务");
    }
    
    // CRON 表达式（每天 10 点执行）
    @Scheduled(cron = "0 0 10 * * *")
    public void taskWithCron() {
        System.out.println("每天 10 点执行");
    }
}
```

### 3. CRON 表达式说明

```
秒  分  小时 日  月  周   年
0   0   10   *   *   *    *

常见示例：
0 0 10 * * *       - 每天 10 点
0 0 10 ? * MON    - 每周一 10 点
0 0 10 1 * *      - 每月 1 号 10 点
0 0/30 * * * *    - 每 30 分钟
0 0 10 * * * 2024 - 仅 2024 年的 10 点
```

---

## 💡 实现细节

### 1. 线程池配置

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {
    
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskScheduler());
    }
    
    @Bean
    public Executor taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);  // 线程池大小
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(60);
        scheduler.initialize();
        return scheduler;
    }
}
```

### 2. 异常处理

```java
@Component
public class SafeScheduledTasks {
    
    @Scheduled(fixedRate = 5000)
    public void safeTask() {
        try {
            // 任务逻辑
        } catch (Exception e) {
            // 记录异常，不中断定时执行
            log.error("定时任务异常", e);
        }
    }
}
```

### 3. 动态启禁用

```java
@Component
public class ConditionalScheduledTask {
    
    @Autowired
    private ConfigProperties config;
    
    @Scheduled(fixedRate = 5000)
    public void conditionalTask() {
        // 根据配置决定是否执行
        if (!config.isScheduleEnabled()) {
            return;
        }
        
        // 执行任务
    }
}
```

---

## 🏢 行业应用

### 数据同步定时任务

```java
@Component
public class DataSyncTasks {
    
    @Autowired
    private DataSyncService syncService;
    
    @Scheduled(cron = "0 0 2 * * *")  // 每天凌晨 2 点
    public void syncData() {
        syncService.syncUserData();
        syncService.syncOrderData();
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| @Scheduled | 定时任务注解 |
| fixedDelay | 固定延迟 |
| fixedRate | 固定速率 |
| cron | CRON 表达式 |
| @EnableScheduling | 启用定时任务 |

---

**恭喜！** 🎉 你已经掌握了简单定时任务！
