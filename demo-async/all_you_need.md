# 异步任务 - 完全学习指南

## 📚 简介

在 Web 应用中，某些操作（如发送邮件、生成报告、调用第三方服务）往往耗时较长。使用异步任务可以快速响应用户，在后台异步处理这些耗时操作。Spring Boot 提供了简单的 @Async 注解支持异步编程。

### 为什么学习这个？
- 🎯 **快速响应：不阻塞用户请求**
- 🎯 **性能提升：充分利用 CPU 多核**
- 🎯 **灵活调度：支持异步和串行执行**
- 🎯 **易于集成：最小化代码修改**

---

## 🎯 核心概念

### 1. 同步 vs 异步

```
同步执行：
用户请求 → 耗时操作 → 返回结果 (总耗时 = 操作耗时)
缺点：用户等待时间长

异步执行：
用户请求 → 后台执行耗时操作 → 立即返回 (总耗时 = 0)
         ↓
      返回任务ID，用户可查询状态
```

### 2. @Async 基本使用

```java
@Configuration
@EnableAsync  // 启用异步
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);  // 核心线程数
        executor.setMaxPoolSize(10);  // 最大线程数
        executor.setQueueCapacity(100);  // 队列容量
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

@Service
public class EmailService {
    
    // 无返回值的异步方法
    @Async("taskExecutor")
    public void sendEmail(String to, String subject, String body) {
        try {
            Thread.sleep(3000);  // 模拟耗时操作
            log.info("邮件已发送给: {}", to);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    // 有返回值的异步方法
    @Async("taskExecutor")
    public Future<String> generateReport() {
        try {
            Thread.sleep(5000);  // 模拟报告生成
            return AsyncResult.forValue("报告已生成");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return AsyncResult.forExecutionException(e);
        }
    }
    
    // 使用 CompletableFuture（更推荐）
    @Async("taskExecutor")
    public CompletableFuture<String> generateReportAsync() {
        try {
            Thread.sleep(5000);
            return CompletableFuture.completedFuture("报告已生成");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
}

// 使用
@RestController
public class EmailController {
    
    @Autowired
    private EmailService emailService;
    
    @GetMapping("/send-email")
    public ApiResponse sendEmail() {
        // 立即返回，邮件在后台发送
        emailService.sendEmail("user@example.com", "Hello", "This is a test");
        return ApiResponse.success("邮件已排队，将在后台发送");
    }
    
    @GetMapping("/generate-report")
    public CompletableFuture<String> generateReport() {
        // 返回 CompletableFuture，调用者可以异步获取结果
        return emailService.generateReportAsync();
    }
}
```

### 3. Future 和 CompletableFuture

```java
@Service
public class AsyncTaskService {
    
    @Autowired
    private Executor taskExecutor;
    
    // 使用 Future（较旧）
    @Async
    public Future<String> doTaskWithFuture() throws InterruptedException {
        Thread.sleep(2000);
        return new AsyncResult<>("任务完成");
    }
    
    // 使用 CompletableFuture（推荐）
    @Async
    public CompletableFuture<String> doTaskWithCompletableFuture() {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
                return "任务完成";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }, taskExecutor);
    }
}

// 使用 Future
Future<String> future = asyncService.doTaskWithFuture();
String result = future.get();  // 阻塞等待结果

// 使用 CompletableFuture（更灵活）
CompletableFuture<String> cf = asyncService.doTaskWithCompletableFuture();
cf.thenAccept(result -> System.out.println("结果: " + result));  // 异步处理结果
cf.thenApply(result -> result.toUpperCase());  // 链式操作
cf.handle((result, ex) -> ex != null ? "失败" : result);  // 错误处理
```

### 4. 异常处理

```java
@Service
@Slf4j
public class AsyncExceptionHandling {
    
    @Async
    public CompletableFuture<String> taskWithException() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("异步任务执行出错");
        }).exceptionally(ex -> {
            log.error("异步任务失败", ex);
            return "错误已处理";
        });
    }
    
    @Async
    public void taskWithCustomHandler() {
        try {
            // 业务逻辑
        } catch (Exception e) {
            log.error("异步任务出错", e);
            // 记录日志、发送告警等
        }
    }
}

// 全局异常处理器
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步方法 {} 执行异常", method.getName(), ex);
            // 发送告警、记录日志等
        };
    }
}
```

---

## 💡 实现细节

### 1. 线程池配置优化

```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean(name = "ioTaskExecutor")
    public Executor ioTaskExecutor() {
        // I/O 密集型任务（如网络请求、数据库操作）
        // 线程数 = CPU 核数 * 2
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors() * 2);
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 3);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("io-executor-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "cpuTaskExecutor")
    public Executor cpuTaskExecutor() {
        // CPU 密集型任务（如计算、数据处理）
        // 线程数 = CPU 核数 + 1
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors() + 1);
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() + 2);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("cpu-executor-");
        executor.initialize();
        return executor;
    }
}
```

### 2. 异步结合消息队列

```java
@Service
public class EmailServiceWithMQ {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    // 快速返回，邮件发送委托给消息队列异步处理
    public void sendEmailAsync(String to, String subject, String body) {
        EmailRequest request = new EmailRequest(to, subject, body);
        rabbitTemplate.convertAndSend("email-queue", request);
    }
}

@Component
public class EmailConsumer {
    
    @RabbitListener(queues = "email-queue")
    public void consumeEmail(EmailRequest request) {
        // 异步发送邮件
        sendEmail(request.getTo(), request.getSubject(), request.getBody());
    }
}
```

---

## 🏢 行业应用

### 1. 用户注册流程

```
用户提交 → 保存用户 → 立即返回 ← 用户看到成功提示
              ↓
         后台异步：发送验证邮件、初始化用户数据、记录日志
```

### 2. 大数据导出

```
用户点击导出 → 生成任务 → 立即返回任务ID
                  ↓
            后台异步生成文件
                  ↓
            用户查询任务状态，得到下载链接
```

---

## 🚀 快速开始

### 1. 启用异步

```java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2. 创建异步方法

```java
@Service
public class AsyncService {
    
    @Async
    public CompletableFuture<String> doAsyncTask() {
        return CompletableFuture.completedFuture("Done");
    }
}
```

### 3. 使用异步方法

```java
@RestController
public class Controller {
    
    @Autowired
    private AsyncService asyncService;
    
    @GetMapping("/task")
    public CompletableFuture<String> startTask() {
        return asyncService.doAsyncTask();
    }
}
```

---

## 🧪 测试

```java
@SpringBootTest
class AsyncTest {
    
    @Autowired
    private AsyncService asyncService;
    
    @Test
    public void testAsync() throws Exception {
        CompletableFuture<String> future = asyncService.doAsyncTask();
        String result = future.get();  // 等待完成
        assertEquals("Done", result);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| @Async | 异步方法注解 |
| @EnableAsync | 启用异步 |
| Executor | 线程执行器 |
| Future | 异步结果 |
| CompletableFuture | 更灵活的异步结果 |
| ThreadPool | 线程池管理 |
| 异常处理 | AsyncUncaughtExceptionHandler |

---

**恭喜！** 🎉 你已经掌握了异步任务编程！
