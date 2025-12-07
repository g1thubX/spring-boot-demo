# 单机限流（Guava）- 完全学习指南

## 📚 简介

限流是保护系统的重要措施。Guava RateLimiter 提供了简单易用的本地限流方案，适合单机应用或每个节点限流的场景。

### 为什么学习这个？
- 🎯 **防护过载：保护系统不被刷垮**
- 🎯 **简单易用：一行代码完成限流**
- 🎯 **性能好：本地无网络开销**
- 🎯 **灵活配置：支持多种限流策略**

---

## 🎯 核心概念

### 1. RateLimiter 基础

```java
// 创建限流器（每秒允许 10 个请求）
RateLimiter rateLimiter = RateLimiter.create(10);

// 获取许可（阻塞直到获得）
rateLimiter.acquire();

// 尝试获取许可（不阻塞）
if (rateLimiter.tryAcquire()) {
    // 处理请求
} else {
    // 限流
    throw new RateLimitException("请求过于频繁");
}

// 获取多个许可
rateLimiter.acquire(5);  // 获取 5 个许可
```

### 2. 在 Spring Boot 中使用

```java
@Configuration
public class RateLimiterConfig {
    
    // API 限流器
    @Bean("apiRateLimiter")
    public RateLimiter apiRateLimiter() {
        return RateLimiter.create(100);  // 每秒 100 个请求
    }
    
    // 登录限流器（更严格）
    @Bean("loginRateLimiter")
    public RateLimiter loginRateLimiter() {
        return RateLimiter.create(5);  // 每秒最多 5 次登录尝试
    }
}

@RestController
@RequestMapping("/users")
public class UserController {
    
    @Autowired
    @Qualifier("apiRateLimiter")
    private RateLimiter rateLimiter;
    
    @GetMapping
    public List<User> list() {
        // 请求获取许可
        if (!rateLimiter.tryAcquire()) {
            throw new RateLimitException("请求过于频繁，请稍后再试");
        }
        
        return userService.list();
    }
}
```

### 3. AOP 方式实现限流

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RateLimit {
    String value() default "default";  // 限流器名称
    double permitsPerSecond() default 10;  // 每秒许可数
}

@Aspect
@Component
public class RateLimitAspect {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Before("@annotation(rateLimit)")
    public void checkRateLimit(JoinPoint jp, RateLimit rateLimit) 
            throws Throwable {
        String name = rateLimit.value();
        double permitsPerSecond = rateLimit.permitsPerSecond();
        
        RateLimiter limiter = limiters.computeIfAbsent(name, 
            k -> RateLimiter.create(permitsPerSecond));
        
        if (!limiter.tryAcquire()) {
            throw new RateLimitException("请求过于频繁");
        }
    }
}

// 使用
@RestController
public class UserController {
    
    @GetMapping("/users")
    @RateLimit(value = "user-list", permitsPerSecond = 100)
    public List<User> list() {
        return userService.list();
    }
    
    @PostMapping("/login")
    @RateLimit(value = "login", permitsPerSecond = 5)
    public ApiResponse login(String username, String password) {
        return userService.login(username, password);
    }
}
```

### 4. 处理限流异常

```java
@RestControllerAdvice
public class RateLimitExceptionHandler {
    
    @ExceptionHandler(RateLimitException.class)
    public ResponseEntity<ApiResponse> handleRateLimit(RateLimitException e) {
        ApiResponse response = ApiResponse.error(429, "请求过于频繁，请稍后再试");
        return ResponseEntity.status(429).body(response);
    }
}
```

---

## 💡 实现细节

### 1. 不同的限流策略

```java
@Component
public class RateLimiterManager {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    // 令牌桶限流
    public RateLimiter createTokenBucketLimiter(String name, double rate) {
        return limiters.computeIfAbsent(name, k -> RateLimiter.create(rate));
    }
    
    // 获取许可，带超时
    public boolean tryAcquire(String name, int permits, long timeout, TimeUnit unit) {
        RateLimiter limiter = limiters.get(name);
        if (limiter == null) return false;
        
        return limiter.tryAcquire(permits, timeout, unit);
    }
    
    // 预热限流器（逐步达到指定速率）
    public RateLimiter createWarmingLimiter(String name, double permitsPerSecond, 
                                           long warmupPeriod, TimeUnit unit) {
        return RateLimiter.create(permitsPerSecond, warmupPeriod, unit);
    }
}
```

### 2. 分布式限流配合

```java
@Service
public class HybridRateLimiterService {
    
    @Autowired
    private RateLimiter localLimiter;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    // 本地 + Redis 双重限流
    public boolean checkLimit(String userId) {
        // 本地限流（快速过滤）
        if (!localLimiter.tryAcquire()) {
            return false;
        }
        
        // Redis 限流（分布式限流）
        String key = "user:rate:" + userId;
        long count = redisTemplate.increment(key);
        if (count == 1) {
            redisTemplate.expire(key, 1, TimeUnit.SECONDS);
        }
        
        return count <= 100;  // 每秒最多 100 个请求
    }
}
```

---

## 🏢 行业应用

### 1. API 网关限流

```java
@Component
public class ApiGatewayRateLimit {
    
    private final RateLimiter globalLimiter = RateLimiter.create(10000);  // 全局
    private final Map<String, RateLimiter> userLimiters = new ConcurrentHashMap<>();  // 用户级
    
    public boolean checkLimit(String userId) {
        // 全局限流
        if (!globalLimiter.tryAcquire()) {
            return false;
        }
        
        // 用户级限流
        RateLimiter userLimiter = userLimiters.computeIfAbsent(userId,
            k -> RateLimiter.create(100));
        
        return userLimiter.tryAcquire();
    }
}
```

### 2. 认证限流

```java
@PostMapping("/login")
@RateLimit(value = "login", permitsPerSecond = 5)  // 5 次/秒
public ApiResponse login() {
    // 防止暴力破解
}
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
</dependency>
```

### 2. 创建限流器

```java
RateLimiter limiter = RateLimiter.create(10);  // 10 请求/秒
```

### 3. 使用

```java
if (limiter.tryAcquire()) {
    // 处理请求
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| RateLimiter | 限流器 |
| 令牌桶 | 限流算法 |
| tryAcquire | 尝试获取许可 |
| acquire | 获取许可（阻塞） |
| 预热 | 逐步达到速率 |

---

**恭喜！** 🎉 你已经掌握了单机限流！
