# Ehcache 本地缓存 - 完全学习指南

## 📚 简介

Ehcache 是一个纯 Java 的开源缓存框架，用于本地进程内的缓存。相比 Redis，Ehcache 无需网络开销，速度更快；相比 Redis，Ehcache 不支持分布式。适合单机应用或缓存热数据。

### 为什么学习这个？
- 🎯 **超快速：本地内存，无网络延迟**
- 🎯 **简单易用：配置简洁，易于集成**
- 🎯 **灵活过期：支持多种过期策略**
- 🎯 **持久化：支持硬盘存储**

---

## 🎯 核心概念

### 1. Ehcache vs Redis

| 特性 | Ehcache | Redis |
|-----|---------|-------|
| 存储位置 | 本地内存 | 远程服务器 |
| 速度 | 超快 | 快 |
| 分布式 | 否 | 是 |
| 内存大小 | 受 JVM 限制 | 可很大 |
| 持久化 | 支持 | 支持 |
| 应用场景 | 单机热数据 | 分布式缓存 |

### 2. 基本使用

```java
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>

// 配置
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new EhCacheCacheManager(ehCacheManagerFactory().getObject());
    }
    
    @Bean
    public EhCacheManagerFactoryBean ehCacheManagerFactory() {
        EhCacheManagerFactoryBean cmf = new EhCacheManagerFactoryBean();
        cmf.setConfigLocation(new ClassPathResource("ehcache.xml"));
        cmf.setShared(true);
        return cmf;
    }
}

// 使用 Spring Cache 注解
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        // 缓存命中返回缓存，未命中执行方法并缓存结果
        return userRepository.findById(id).orElse(null);
    }
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        // 执行方法并更新缓存
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        // 删除缓存
        userRepository.deleteById(id);
    }
}
```

### 3. Ehcache 配置文件

```xml
<!-- ehcache.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd">
    
    <!-- 磁盘缓存位置 -->
    <diskStore path="java.io.tmpdir/ehcache"/>
    
    <!-- 默认缓存配置 -->
    <defaultCache maxEntriesLocalHeap="10000"
                  eternal="false"
                  timeToIdleSeconds="300"
                  timeToLiveSeconds="600"
                  overflowToDisk="true"
                  statistics="true">
    </defaultCache>
    
    <!-- users 缓存配置 -->
    <cache name="users"
           maxEntriesLocalHeap="5000"
           eternal="false"
           timeToIdleSeconds="600"
           timeToLiveSeconds="3600"
           overflowToDisk="true"
           statistics="true"/>
    
    <!-- 永不过期的缓存 -->
    <cache name="config"
           maxEntriesLocalHeap="1000"
           eternal="true"
           overflowToDisk="false"/>
    
    <!-- 参数说明：
         - maxEntriesLocalHeap: 内存中最多缓存条数
         - eternal: 是否永不过期
         - timeToIdleSeconds: 空闲过期时间（秒）
         - timeToLiveSeconds: 生存时间（秒）
         - overflowToDisk: 超出内存是否写入磁盘
         - statistics: 是否记录统计信息
    -->
</ehcache>
```

### 4. 直接使用 Ehcache API

```java
@Service
public class DirectEhcacheService {
    
    private final CacheManager cacheManager;
    
    public DirectEhcacheService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    public void putData(String key, Object value) {
        Cache cache = cacheManager.getCache("users");
        cache.put(new Element(key, value));
    }
    
    public Object getData(String key) {
        Cache cache = cacheManager.getCache("users");
        Element element = cache.get(key);
        return element != null ? element.getObjectValue() : null;
    }
    
    public void removeData(String key) {
        Cache cache = cacheManager.getCache("users");
        cache.remove(key);
    }
    
    public void clearCache() {
        Cache cache = cacheManager.getCache("users");
        cache.removeAll();
    }
}
```

---

## 💡 实现细节

### 1. 自定义缓存过期策略

```java
@Configuration
public class EhcacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CacheConfiguration userCacheConfig = new CacheConfiguration()
            .name("users")
            .maxEntriesLocalHeap(5000)
            .eternal(false)
            .timeToIdleSeconds(300)  // 5 分钟空闲过期
            .timeToLiveSeconds(3600)  // 1 小时最长生存时间
            .overflowToDisk(true);
        
        CacheConfiguration configCacheConfig = new CacheConfiguration()
            .name("config")
            .maxEntriesLocalHeap(100)
            .eternal(true)  // 永不过期，除非手动删除
            .overflowToDisk(false);
        
        Configuration config = new Configuration()
            .defaultCache(new CacheConfiguration()
                .maxEntriesLocalHeap(10000)
                .eternal(false)
                .timeToIdleSeconds(600))
            .cache(userCacheConfig)
            .cache(configCacheConfig);
        
        return CacheManagerBuilder.newCacheManager(config);
    }
}
```

### 2. 缓存预热

```java
@Service
public class CacheWarmupService {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @PostConstruct  // 应用启动时执行
    public void warmupCache() {
        Cache cache = cacheManager.getCache("users");
        
        // 预加载热数据
        List<User> hotUsers = userRepository.findHotUsers();
        hotUsers.forEach(user -> {
            cache.put(new Element("user:" + user.getId(), user));
        });
        
        log.info("缓存预热完成，共加载 {} 条数据", hotUsers.size());
    }
}
```

### 3. 缓存监听和统计

```java
@Service
public class CacheMonitorService {
    
    @Autowired
    private CacheManager cacheManager;
    
    public void printCacheStats() {
        Cache cache = cacheManager.getCache("users");
        
        CacheStatistics stats = cache.getStatistics();
        
        System.out.println("缓存命中: " + stats.getCacheHits());
        System.out.println("缓存未命中: " + stats.getCacheMisses());
        System.out.println("缓存命中率: " + 
            String.format("%.2f%%", 
                (double) stats.getCacheHits() / 
                (stats.getCacheHits() + stats.getCacheMisses()) * 100));
        System.out.println("缓存大小: " + cache.getSize());
    }
}
```

---

## 🏢 行业应用

### 1. 热门数据缓存

```java
@Service
public class HotDataCacheService {
    
    // 缓存热门商品
    @Cacheable(value = "hotProducts", key = "'all'")
    public List<Product> getHotProducts() {
        return productRepository.findHotProducts();
    }
    
    // 缓存配置数据
    @Cacheable(value = "config", key = "#configKey")
    public String getConfig(String configKey) {
        return configRepository.findByKey(configKey).getValue();
    }
}
```

### 2. 二级缓存（Ehcache + Redis）

```
应用 → Ehcache（一级缓存）→ Redis（二级缓存）→ 数据库
       本地，超快             分布式，快          最慢
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

### 2. 配置

在 `src/main/resources` 下创建 `ehcache.xml` 配置文件。

### 3. 启用缓存

```java
@SpringBootApplication
@EnableCaching
public class Application {
}
```

---

## 🧪 测试

```java
@SpringBootTest
class EhcacheTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    public void testCacheHit() {
        // 第一次调用，查询数据库
        userService.getUser(1L);
        
        // 第二次调用，从缓存返回
        userService.getUser(1L);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Ehcache | 本地缓存框架 |
| CacheManager | 缓存管理器 |
| Cache | 缓存实例 |
| Element | 缓存元素 |
| @Cacheable | 读缓存 |
| @CachePut | 写缓存 |
| @CacheEvict | 删除缓存 |

---

**恭喜！** 🎉 你已经掌握了 Ehcache 本地缓存！
