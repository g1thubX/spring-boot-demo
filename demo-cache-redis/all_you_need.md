# Redis 缓存 - 完全学习指南

## 📚 简介

Redis 是一个开源的、基于内存的数据结构存储库，常用作缓存、消息队列和会话存储。Spring Boot 集成 Redis 后，可以轻松实现应用级别的缓存，显著提升应用性能。

### 为什么学习这个？
- 🎯 **理解缓存的原理和设计模式**
- 🎯 **掌握 Redis 基础操作**
- 🎯 **学会使用 Spring Cache 注解简化缓存操作**
- 🎯 **解决常见的缓存问题（穿透、击穿、雪崩）**

### 核心价值
- **性能提升**：10-100 倍的读取速度提升
- **减少数据库压力**：大量查询走缓存，而不是数据库
- **分布式锁**：处理并发问题
- **会话共享**：支持分布式会话管理

---

## 🎯 核心概念详解

### 1. 缓存的基本原理

```
请求流程：
┌─────────────┐
│  请求数据   │
└──────┬──────┘
       │
       ├─→ Redis 中是否存在？
       │      │
       │      ├─ 存在 (Cache Hit) ─→ 直接返回 ✓ 快速
       │      │
       │      └─ 不存在 (Cache Miss) ─→ 查询数据库
       │                                  │
       │                                  ├─ 取得数据
       │                                  │
       │                                  └─ 存入 Redis 缓存
       │
       └─→ 返回数据给客户端
```

**缓存命中率 = 缓存命中次数 / 总请求次数**

典型场景：
- 热点数据（Top 100 商品）缓存命中率 > 95%
- 用户信息缓存命中率 > 80%
- 搜索结果缓存命中率 > 50%

### 2. Redis 基本数据结构

```
String（字符串）
  └─ 最常用，存储 JSON 或简单值
     例: key="user:1" value='{"id":1,"name":"Alice"}'

Hash（哈希表）
  └─ 存储对象，比 String 更节省空间
     例: hset("user:1", "name", "Alice")
            hset("user:1", "email", "alice@example.com")

List（列表）
  └─ 存储队列、栈、时间线
     例: lpush("user:messages", "msg1", "msg2")

Set（集合）
  └─ 存储无序集合，支持交集、并集、差集
     例: sadd("user:1:tags", "java", "python")

Sorted Set（有序集合）
  └─ 存储排序的数据，如排行榜
     例: zadd("game:ranking", 100, "player1")
```

### 3. Spring Boot 集成 Redis

**两种使用方式：**

#### a) RedisTemplate（底层 API）
```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // 设置序列化方式
        StringRedisSerializer stringSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer<?> jackson2JsonRedisSerializer = 
            new Jackson2JsonRedisSerializer<>(Object.class);
        
        template.setKeySerializer(stringSerializer);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        
        return template;
    }
}

@Service
public class UserService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void cacheUser(User user) {
        // 存储
        redisTemplate.opsForValue().set("user:" + user.getId(), user, Duration.ofHours(1));
    }
    
    public User getUserFromCache(Long userId) {
        // 读取
        return (User) redisTemplate.opsForValue().get("user:" + userId);
    }
}
```

#### b) Spring Cache（推荐）
```java
@Configuration
@EnableCaching  // 启用缓存
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.create(factory);
    }
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    // 首次调用时执行方法，结果存入缓存
    // 后续调用直接从缓存返回
    @Cacheable(value = "users", key = "#userId")
    public User getUser(Long userId) {
        log.info("查询数据库...");
        return userRepository.findById(userId).orElse(null);
    }
    
    // 更新时同时更新缓存
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        log.info("更新数据库...");
        return userRepository.save(user);
    }
    
    // 删除时清除缓存
    @CacheEvict(value = "users", key = "#userId")
    public void deleteUser(Long userId) {
        log.info("删除数据库...");
        userRepository.deleteById(userId);
    }
    
    // 清除所有 users 缓存
    @CacheEvict(value = "users", allEntries = true)
    public void clearAllUserCache() {
        log.info("清空所有用户缓存...");
    }
}
```

### 4. Spring Cache 核心注解

| 注解 | 功能 | 场景 |
|-----|------|------|
| @Cacheable | 查询时使用缓存 | 查询、获取数据 |
| @CachePut | 无论如何都执行方法，更新缓存 | 更新数据 |
| @CacheEvict | 删除缓存 | 删除数据、刷新缓存 |
| @Caching | 组合多个注解 | 复杂的缓存操作 |

### 5. 缓存键生成策略

```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;
    
    // 策略 1: 使用参数作为 key
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(Long productId) {
        return productRepository.findById(productId).orElse(null);
    }
    
    // 策略 2: 组合多个参数作为 key
    @Cacheable(value = "products", key = "#categoryId + ':' + #productId")
    public Product getProductByCategory(Long categoryId, Long productId) {
        return productRepository.findByIdAndCategoryId(productId, categoryId).orElse(null);
    }
    
    // 策略 3: 使用 SpEL 表达式
    @Cacheable(value = "products", key = "T(com.example.utils.CacheKeyUtils).buildKey(#productId)")
    public Product getProductWithCustomKey(Long productId) {
        return productRepository.findById(productId).orElse(null);
    }
    
    // 策略 4: 使用对象的属性
    @Cacheable(value = "products", key = "#product.id")
    public void cacheProduct(Product product) {
        // ...
    }
}
```

---

## 💡 实现细节深入分析

### 1. 缓存一致性问题

**问题：** 数据库改变，但缓存未更新

```
场景 1: 旁路缓存（Cache-Aside）
1. 查询时先查缓存
2. 缓存未命中则查询数据库
3. 将数据存入缓存
4. 更新时先更新数据库，再删除缓存
5. 下次查询时重新加载到缓存

优点: 简单，适合读多写少
缺点: 可能出现短暂的不一致

场景 2: 写穿缓存（Write-Through）
1. 更新时同时更新数据库和缓存
2. 确保两者一致
3. 性能较低

场景 3: 写回缓存（Write-Behind）
1. 更新时只更新缓存
2. 异步同步到数据库
3. 性能高，但可能丢失数据
```

### 2. 使用 Spring Cache 管理缓存一致性

```java
@Service
@Transactional
public class ProductService {
    @Autowired
    private ProductRepository productRepository;
    
    // 查询时使用缓存
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElse(null);
    }
    
    // 保存时更新缓存
    @CachePut(value = "products", key = "#result.id")
    public Product saveProduct(Product product) {
        return productRepository.save(product);
    }
    
    // 更新时更新缓存
    @Transactional
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
    
    // 删除时清除缓存
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

### 3. 缓存击穿、穿透、雪崩问题

```
缓存击穿（Cache Breakdown）
├─ 原因: 热点 key 过期，大量请求击穿到数据库
├─ 表现: 数据库负载陡增
└─ 解决: 
   ├─ 使用分布式锁
   ├─ 设置永不过期（需定期更新）
   └─ 热点 key 单独处理

缓存穿透（Cache Penetration）
├─ 原因: 查询不存在的 key，每次都查数据库
├─ 表现: 数据库压力大
└─ 解决:
   ├─ 缓存空值
   ├─ 使用 Bloom Filter（布隆过滤器）
   └─ 参数验证

缓存雪崩（Cache Avalanche）
├─ 原因: 大量 key 同时过期
├─ 表现: 数据库宕机
└─ 解决:
   ├─ 随机过期时间
   ├─ 业务降级
   └─ 缓存预热
```

### 4. 分布式锁实现

```java
@Configuration
public class RedisLockConfig {
    @Bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }
}

@Component
public class DistributedLock {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final long DEFAULT_TIMEOUT = 30;  // 秒
    
    /**
     * 获取锁
     */
    public boolean lock(String key, String value, long timeout) {
        Boolean success = redisTemplate.opsForValue().setIfAbsent(
            key, 
            value, 
            Duration.ofSeconds(timeout)
        );
        return success != null && success;
    }
    
    /**
     * 释放锁（防止删除别人的锁）
     */
    public boolean unlock(String key, String value) {
        String currentValue = redisTemplate.opsForValue().get(key);
        if (value.equals(currentValue)) {
            return redisTemplate.delete(key);
        }
        return false;
    }
}

// 使用分布式锁防止缓存击穿
@Service
public class ProductService {
    @Autowired
    private DistributedLock lock;
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public Product getHotProduct(Long id) {
        String cacheKey = "product:" + id;
        
        // 先查缓存
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return JSON.parseObject(cached, Product.class);
        }
        
        // 缓存不存在，尝试获取锁
        String lockKey = "lock:" + id;
        String lockValue = UUID.randomUUID().toString();
        
        if (lock.lock(lockKey, lockValue, 30)) {
            try {
                // 再次查缓存（其他线程可能已加载）
                cached = redisTemplate.opsForValue().get(cacheKey);
                if (cached != null) {
                    return JSON.parseObject(cached, Product.class);
                }
                
                // 从数据库加载
                Product product = loadFromDatabase(id);
                
                // 存入缓存
                redisTemplate.opsForValue().set(
                    cacheKey, 
                    JSON.toJSONString(product),
                    Duration.ofHours(1)
                );
                
                return product;
            } finally {
                // 释放锁
                lock.unlock(lockKey, lockValue);
            }
        } else {
            // 没有获取到锁，等待后重试
            Thread.sleep(50);
            return getHotProduct(id);
        }
    }
}
```

---

## 🏢 行业应用举例

### 1. 电商系统商品缓存

```java
@Service
public class ProductCacheService {
    @Cacheable(value = "product", key = "#productId", unless = "#result == null")
    public ProductDTO getProduct(Long productId) {
        // 查询商品信息、价格、库存等
        return productRepository.getProductDTO(productId);
    }
    
    @CacheEvict(value = "product", key = "#productId")
    public void updateProductPrice(Long productId, BigDecimal newPrice) {
        productRepository.updatePrice(productId, newPrice);
    }
}
```

### 2. 用户会话缓存

```yaml
spring:
  session:
    store-type: redis
  redis:
    host: localhost
    port: 6379
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class RedisSessionConfig {
}
```

### 3. 排行榜缓存

```java
@Service
public class RankingService {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 增加玩家得分
    public void addScore(String playerId, double score) {
        redisTemplate.opsForZSet().incrementScore("game:ranking", playerId, score);
    }
    
    // 获取排行榜
    public List<String> getTopPlayers(int limit) {
        Set<String> players = redisTemplate.opsForZSet()
            .reverseRange("game:ranking", 0, limit - 1);
        return new ArrayList<>(players);
    }
}
```

---

## 🚀 快速开始指南

### 1. 环境配置

**pom.xml：**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- 使用 Lettuce 客户端（默认） -->
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
</dependency>
```

**application.yml：**
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: ''
    timeout: 2000ms
    jedis:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
  cache:
    type: redis
    redis:
      time-to-live: 3600000  # 1小时
```

### 2. 启用缓存

```java
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 3. 使用缓存注解

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
}
```

---

## 🧪 详细测试指南

### 测试 1：缓存是否有效

```bash
# 1. 启动 Redis
redis-server

# 2. 启动应用
mvn spring-boot:run

# 3. 第一次请求（会查询数据库）
curl http://localhost:8080/users/1

# 4. 第二次请求（会从缓存返回）
curl http://localhost:8080/users/1

# 查看 Redis
redis-cli
> KEYS "*"
> GET "users::1"
```

### 测试 2：缓存过期

```java
@Test
void testCacheExpire() throws Exception {
    userService.getUser(1L);
    
    assertTrue(redisTemplate.hasKey("users::1"));
    
    Thread.sleep(4000);  // 等待 3 秒超时
    
    assertFalse(redisTemplate.hasKey("users::1"));
}
```

### 测试 3：缓存更新

```java
@Test
void testCacheUpdate() {
    User user = userService.getUser(1L);
    assertEquals("Original Name", user.getName());
    
    user.setName("Updated Name");
    userService.updateUser(user);
    
    User cached = userService.getUser(1L);
    assertEquals("Updated Name", cached.getName());
}
```

---

## 🛠️ 开发实践指南

### 实践 1：自定义缓存配置

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new Jackson2JsonRedisSerializer<>(Object.class)));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

### 实践 2：缓存预热

```java
@Component
public class CacheWarmup {
    @Autowired
    private UserService userService;
    
    @PostConstruct
    public void warmup() {
        // 应用启动时预热热点数据
        List<Long> hotUserIds = List.of(1L, 2L, 3L, 4L, 5L);
        hotUserIds.forEach(userService::getUser);
        log.info("缓存预热完成");
    }
}
```

### 实践 3：监控缓存统计

```java
@Component
public class CacheStats {
    @Autowired
    private CacheManager cacheManager;
    
    @Scheduled(fixedRate = 60000)
    public void printCacheStats() {
        cacheManager.getCacheNames().forEach(name -> {
            Cache cache = cacheManager.getCache(name);
            log.info("缓存 {} 的统计信息: {}", name, cache.getNativeCache());
        });
    }
}
```

---

## 📊 常见问题解决

### Q1: 如何选择合适的过期时间？
**A:** 
- 热点数据：1-2 小时
- 普通数据：30 分钟
- 用户信息：1 小时
- 配置数据：永不过期或定期更新

### Q2: 缓存穿透怎么办？
**A:** 缓存空值
```java
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public User getUser(Long id) {
    return userRepository.findById(id).orElse(null);
}

// 或主动缓存空值
public User getUserWithNullCache(Long id) {
    User user = userRepository.findById(id).orElse(null);
    if (user == null) {
        // 缓存空对象，防止穿透
        cache.set("user:" + id, new NullObject(), Duration.ofMinutes(5));
    }
    return user;
}
```

### Q3: Redis 内存爆炸怎么办？
**A:**
1. 设置合理的过期时间
2. 使用 Redis 的 maxmemory 和 eviction 策略
3. 定期清理过期数据
4. 监控内存使用

---

## 📖 扩展学习路径

1. **Redis Cluster** → 集群部署
2. **Redis 哨兵** → 高可用
3. **缓存预热** → 应用启动时加载
4. **一致性哈希** → 分布式缓存
5. **demo-cache-ehcache** → 本地缓存

---

**恭喜！** 🎉 你已经掌握了 Redis 缓存的核心知识。正确使用缓存能将应用性能提升 10 倍以上！
