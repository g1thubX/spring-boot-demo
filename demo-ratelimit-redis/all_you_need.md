# 分布式限流（Redis）- 完全学习指南

## 📚 简介

使用 Redis + Lua 脚本实现分布式限流，支持多个服务器实例共用一个限流策略，比单机限流更加可靠。

### 核心特点
- 🎯 **分布式：多个服务器共享限流策略**
- 🎯 **原子性：Lua 脚本保证原子操作**
- 🎯 **高效：Redis 内存操作，性能好**

---

## 🎯 实现方法

### 1. 使用 Lua 脚本限流

```java
@Configuration
public class DistributedRateLimiterConfig {
    
    @Bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }
}

@Component
public class DistributedRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private final static Long LIMIT_SUCCESS = 1L;
    
    // Lua 脚本：原子性地检查和更新计数
    private final String LUA_SCRIPT = 
        "if redis.call('exists', KEYS[1]) == 1 then\n" +
        "    local count = redis.call('incr', KEYS[1])\n" +
        "    if count > tonumber(ARGV[1]) then\n" +
        "        return 0\n" +
        "    end\n" +
        "    return 1\n" +
        "else\n" +
        "    redis.call('setex', KEYS[1], tonumber(ARGV[2]), 1)\n" +
        "    return 1\n" +
        "end";
    
    public boolean tryAcquire(String key, int permits, int timeoutSec) {
        // key: 限流键
        // permits: 单位时间内允许的请求数
        // timeoutSec: 时间窗口大小（秒）
        
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(
            LUA_SCRIPT, Long.class);
        
        Long result = redisTemplate.execute(
            redisScript,
            Collections.singletonList(key),
            String.valueOf(permits),
            String.valueOf(timeoutSec)
        );
        
        return result != null && result.equals(LIMIT_SUCCESS);
    }
}

// 使用
@RestController
public class UserController {
    
    @Autowired
    private DistributedRateLimiter rateLimiter;
    
    @GetMapping("/users")
    public List<User> list() {
        // 每个用户 IP 每秒最多 100 个请求
        String key = "rate:user:" + getClientIp();
        
        if (!rateLimiter.tryAcquire(key, 100, 1)) {
            throw new RateLimitException("请求过于频繁");
        }
        
        return userService.list();
    }
}
```

### 2. 滑动窗口限流

```java
@Component
public class SlidingWindowRateLimiter {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public boolean tryAcquire(String key, int limit, int windowSeconds) {
        long now = System.currentTimeMillis();
        long start = now - windowSeconds * 1000;
        
        // 移除过期的请求记录
        redisTemplate.opsForZSet().removeRangeByScore(key, 0, start);
        
        // 统计时间窗口内的请求数
        Long count = redisTemplate.opsForZSet().count(key, start, now);
        
        if (count < limit) {
            // 添加当前请求
            redisTemplate.opsForZSet().add(key, UUID.randomUUID().toString(), now);
            // 设置过期时间
            redisTemplate.expire(key, windowSeconds, TimeUnit.SECONDS);
            return true;
        }
        
        return false;
    }
}
```

### 3. 令牌桶限流

```java
@Component
public class TokenBucketRateLimiter {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private final String LUA_SCRIPT =
        "local rate_limit_key = KEYS[1]\n" +
        "local capacity = tonumber(ARGV[1])\n" +
        "local fill_rate = tonumber(ARGV[2])\n" +
        "local now = tonumber(ARGV[3])\n" +
        "local requested = tonumber(ARGV[4])\n" +
        "\n" +
        "local current = redis.call('hgetall', rate_limit_key)\n" +
        "local tokens = capacity\n" +
        "local last_update = now\n" +
        "\n" +
        "if #current > 0 then\n" +
        "    tokens = tonumber(current[2])\n" +
        "    last_update = tonumber(current[4])\n" +
        "    local elapsed = math.max(0, now - last_update)\n" +
        "    tokens = math.min(capacity, tokens + elapsed * fill_rate)\n" +
        "end\n" +
        "\n" +
        "local allowed = tokens >= requested\n" +
        "\n" +
        "if allowed then\n" +
        "    tokens = tokens - requested\n" +
        "end\n" +
        "\n" +
        "redis.call('hset', rate_limit_key, 'tokens', tokens, 'last_update', now)\n" +
        "redis.call('expire', rate_limit_key, 3600)\n" +
        "\n" +
        "return allowed and 1 or 0";
    
    public boolean tryAcquire(String key, double capacity, double fillRate) {
        // capacity: 桶容量
        // fillRate: 每秒填充速率
        
        long now = System.currentTimeMillis() / 1000;
        
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>(
            LUA_SCRIPT, Long.class);
        
        Long result = redisTemplate.execute(
            redisScript,
            Collections.singletonList(key),
            String.valueOf(capacity),
            String.valueOf(fillRate),
            String.valueOf(now),
            "1"
        );
        
        return result != null && result.equals(1L);
    }
}
```

---

## 🏢 行业应用

### 用户级限流

```java
@Service
public class ApiService {
    
    @Autowired
    private DistributedRateLimiter rateLimiter;
    
    public void processRequest(String userId) {
        String key = "user:rate:" + userId;
        
        if (!rateLimiter.tryAcquire(key, 1000, 60)) {  // 每分钟 1000 个请求
            throw new RateLimitException("超出配额");
        }
        
        // 处理请求
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Lua 脚本 | 保证原子性 |
| 滑动窗口 | 更精确的限流 |
| 令牌桶 | 平滑的请求处理 |
| Redis | 分布式存储 |

---

**恭喜！** 🎉 你已经掌握了分布式限流！
