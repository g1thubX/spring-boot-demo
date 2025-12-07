# AOP 日志记录 - 完全学习指南

## 📚 简介

使用 AOP（面向切面编程）记录 Web 请求的详细信息（参数、返回值、执行时间等），可以快速定位问题，优化性能。

### 核心特点
- 🎯 **自动记录：无需修改业务代码**
- 🎯 **完整信息：捕获请求、响应、异常、耗时**
- 🎯 **易于维护：集中式日志管理**
- 🎯 **性能可观：可分析慢查询**

---

## 🎯 使用方法

### 1. 创建 AOP 切面

```java
@Aspect
@Component
@Slf4j
public class WebLogAspect {
    
    // 切点：所有 Controller 方法
    @Pointcut("execution(public * com.xkcoding.controller.*.*(..))")
    public void webLog() {}
    
    @Before("webLog()")
    public void doBefore(JoinPoint jp) throws Throwable {
        ServletRequestAttributes attributes = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        
        log.info("========== 请求开始 ==========");
        log.info("URL: {}", request.getRequestURL());
        log.info("HTTP 方法: {}", request.getMethod());
        log.info("IP: {}", request.getRemoteAddr());
        log.info("类: {}", jp.getSignature().getDeclaringTypeName());
        log.info("方法: {}", jp.getSignature().getName());
        log.info("参数: {}", Arrays.toString(jp.getArgs()));
    }
    
    @After("webLog()")
    public void doAfter() {
        log.info("========== 请求结束 ==========");
    }
    
    @AfterReturning(pointcut = "webLog()", returning = "ret")
    public void doAfterReturning(Object ret) throws Throwable {
        log.info("返回值: {}", ret);
    }
    
    @AfterThrowing(pointcut = "webLog()", throwing = "ex")
    public void doAfterThrowing(Exception ex) throws Throwable {
        log.error("异常信息: ", ex);
    }
}
```

### 2. 记录执行耗时

```java
@Aspect
@Component
@Slf4j
public class TimingAspect {
    
    private static final String START_TIME = "startTime";
    
    @Pointcut("execution(public * com.xkcoding.controller.*.*(..))")
    public void timing() {}
    
    @Before("timing()")
    public void recordStartTime() {
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        requestAttributes.setAttribute(START_TIME, System.currentTimeMillis(), 
                                     RequestAttributes.SCOPE_REQUEST);
    }
    
    @AfterReturning("timing()")
    public void recordEndTime() {
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        Long startTime = (Long) requestAttributes.getAttribute(START_TIME, 
                                                              RequestAttributes.SCOPE_REQUEST);
        long duration = System.currentTimeMillis() - startTime;
        
        log.info("请求耗时: {} ms", duration);
        
        // 超过 1000ms 记录为慢查询
        if (duration > 1000) {
            log.warn("慢查询告警: 耗时 {} ms", duration);
        }
    }
}
```

### 3. 自定义日志注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @annotation WebLog {
    String value() default "";  // 操作描述
    boolean logParams() default true;  // 是否记录参数
    boolean logResult() default true;  // 是否记录返回值
}

@Aspect
@Component
@Slf4j
public class WebLogAspectWithAnnotation {
    
    @Around("@annotation(webLog)")
    public Object doAround(ProceedingJoinPoint jp, WebLog webLog) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        try {
            log.info("方法: {}, 描述: {}", jp.getSignature().getName(), webLog.value());
            
            if (webLog.logParams()) {
                log.info("参数: {}", JSON.toJSONString(jp.getArgs()));
            }
            
            // 执行方法
            Object result = jp.proceed();
            
            if (webLog.logResult()) {
                log.info("返回值: {}", JSON.toJSONString(result));
            }
            
            return result;
        } catch (Exception ex) {
            log.error("方法异常", ex);
            throw ex;
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("耗时: {} ms", duration);
        }
    }
}

// 使用
@RestController
public class UserController {
    
    @GetMapping("/users")
    @WebLog("查询用户列表")
    public List<User> list() {
        return userService.list();
    }
    
    @PostMapping("/users")
    @WebLog(value = "创建用户", logResult = false)
    public User create(@RequestBody User user) {
        return userService.create(user);
    }
}
```

---

## 💡 实现细节

### 1. 记录请求和响应

```java
@Aspect
@Component
@Slf4j
public class RequestResponseLoggingAspect {
    
    @Around("execution(* com.xkcoding.controller.*.*(..))")
    public Object around(ProceedingJoinPoint jp) throws Throwable {
        ServletRequestAttributes attrs = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attrs.getRequest();
        
        // 读取请求体
        String requestBody = getRequestBody(request);
        
        log.info("请求URL: {}", request.getRequestURL());
        log.info("请求体: {}", requestBody);
        
        // 执行方法
        Object result = jp.proceed();
        
        log.info("响应结果: {}", JSON.toJSONString(result));
        
        return result;
    }
}
```

### 2. 性能分析

```java
@Component
@Slf4j
public class PerformanceAnalyzer {
    
    private final AtomicInteger slowQueryCount = new AtomicInteger(0);
    
    @AfterReturning("execution(* com.xkcoding.repository.*.*(..))")
    public void analyzePerformance(JoinPoint jp) {
        long startTime = (long) jp.proceed();
        long duration = System.currentTimeMillis() - startTime;
        
        if (duration > 100) {  // 100ms 以上认为是慢查询
            slowQueryCount.incrementAndGet();
            log.warn("慢查询: {} 耗时 {}ms", jp.getSignature(), duration);
        }
    }
}
```

---

## 🏢 行业应用

### 审计日志

```java
@Aspect
@Component
@Slf4j
public class AuditAspect {
    
    @Pointcut("execution(* com.xkcoding.service.*.save*(..))" +
             " || execution(* com.xkcoding.service.*.update*(..))" +
             " || execution(* com.xkcoding.service.*.delete*(..))")
    public void audit() {}
    
    @AfterReturning("audit()")
    public void recordAuditLog(JoinPoint jp) {
        String operation = jp.getSignature().getName();
        String details = JSON.toJSONString(jp.getArgs());
        String operator = getCurrentUser();
        LocalDateTime timestamp = LocalDateTime.now();
        
        // 保存审计日志
        auditLogRepository.save(new AuditLog(operator, operation, details, timestamp));
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| @Aspect | 定义切面 |
| @Pointcut | 定义切点 |
| @Before | 前置通知 |
| @After | 后置通知 |
| @Around | 环绕通知 |
| JoinPoint | 连接点信息 |

---

**恭喜！** 🎉 你已经掌握了 AOP 日志记录！
