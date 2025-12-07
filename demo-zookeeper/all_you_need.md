# Zookeeper 分布式锁 - 完全学习指南

## 📚 简介

Zookeeper 是一个分布式协调服务，可用于实现分布式锁、服务注册发现、配置中心等。这个 demo 展示了如何使用 Zookeeper 实现分布式锁，解决多服务器的并发问题。

### 核心特点
- 🎯 **分布式锁：跨多个服务器的互斥锁**
- 🎯 **强一致性：保证数据一致性**
- 🎯 **高可用：支持故障转移**
- 🎯 **简化开发：AOP 自动应用锁**

---

## 🎯 使用方法

### 1. 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.7.0</version>
</dependency>

<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.0.0</version>
</dependency>
```

### 2. Zookeeper 配置

```yaml
zookeeper:
  server: localhost:2181  # Zookeeper 连接地址
  path: /locks  # 锁的根路径
```

### 3. 分布式锁实现

```java
@Configuration
public class ZookeeperConfig {
    
    @Bean
    public CuratorFramework curatorFramework(@Value("${zookeeper.server}") String connectString) {
        CuratorFramework client = CuratorFrameworkFactory.newClient(
            connectString,
            3000,
            3000,
            new ExponentialBackoffRetry(1000, 3)
        );
        client.start();
        return client;
    }
    
    @Bean
    public DistributedLock distributedLock(CuratorFramework client) {
        return new ZookeeperDistributedLock(client);
    }
}

// 分布式锁接口
public interface DistributedLock {
    boolean tryLock(String key, long timeout, TimeUnit unit) throws Exception;
    void unlock(String key) throws Exception;
}

// Zookeeper 实现
@Component
public class ZookeeperDistributedLock implements DistributedLock {
    
    private final CuratorFramework client;
    private final Map<String, InterProcessMutex> locks = new ConcurrentHashMap<>();
    
    public ZookeeperDistributedLock(CuratorFramework client) {
        this.client = client;
    }
    
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) throws Exception {
        String lockPath = "/locks/" + key;
        InterProcessMutex mutex = locks.computeIfAbsent(lockPath, 
            k -> new InterProcessMutex(client, k));
        
        return mutex.acquire(timeout, unit);
    }
    
    @Override
    public void unlock(String key) throws Exception {
        String lockPath = "/locks/" + key;
        InterProcessMutex mutex = locks.get(lockPath);
        if (mutex != null && mutex.isAcquiredInThisProcess()) {
            mutex.release();
        }
    }
}
```

### 4. AOP 自动锁定

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ZooLock {
    String value();  // 锁的键
}

@Aspect
@Component
public class ZooLockAspect {
    
    @Autowired
    private DistributedLock lock;
    
    @Around("@annotation(zooLock)")
    public Object around(ProceedingJoinPoint jp, ZooLock zooLock) throws Throwable {
        String key = zooLock.value();
        
        if (!lock.tryLock(key, 10, TimeUnit.SECONDS)) {
            throw new LockException("获取锁失败: " + key);
        }
        
        try {
            return jp.proceed();
        } finally {
            lock.unlock(key);
        }
    }
}

// 使用
@Service
public class InventoryService {
    
    @ZooLock("product:123")
    public void deductInventory(Long productId, Integer quantity) {
        // 库存扣减，同时只允许一个线程执行
        Product product = productRepository.findById(productId).orElseThrow();
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
    }
}
```

---

## 💡 实现细节

### 1. 可重入锁

```java
// Zookeeper 的 InterProcessMutex 是可重入的
InterProcessMutex mutex = new InterProcessMutex(client, "/locks/key");

// 同一线程可以多次获取锁
mutex.acquire();
mutex.acquire();  // 不会阻塞

// 需要释放相同次数
mutex.release();
mutex.release();
```

### 2. 锁超时处理

```java
@Component
public class LockTimeoutHandler {
    
    @Autowired
    private DistributedLock lock;
    
    public void executeWithTimeout(String key, Callable<Void> task, long timeoutMs) {
        try {
            if (!lock.tryLock(key, timeoutMs, TimeUnit.MILLISECONDS)) {
                throw new LockTimeoutException("获取锁超时: " + key);
            }
            
            task.call();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            try {
                lock.unlock(key);
            } catch (Exception e) {
                log.error("释放锁失败", e);
            }
        }
    }
}
```

### 3. 监控锁状态

```java
@Component
@Scheduled(fixedRate = 60000)
public class LockMonitor {
    
    @Autowired
    private CuratorFramework client;
    
    public void monitorLocks() throws Exception {
        List<String> locks = client.getChildren().forPath("/locks");
        log.info("当前锁数: {}", locks.size());
        locks.forEach(lock -> log.debug("锁: {}", lock));
    }
}
```

---

## 🏢 行业应用

### 库存控制

```java
@Service
public class InventoryService {
    
    @ZooLock("inventory:#{#productId}")
    public void deductInventory(Long productId, Integer quantity) {
        // 扣减库存，保证原子性
        Product product = productRepository.findById(productId).orElseThrow();
        if (product.getStock() < quantity) {
            throw new InsufficientInventoryException();
        }
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| CuratorFramework | Zookeeper 客户端 |
| InterProcessMutex | 分布式互斥锁 |
| 分布式锁 | 跨进程的互斥 |
| 可重入 | 同线程可重复获取 |

---

**恭喜！** 🎉 你已经掌握了 Zookeeper 分布式锁！
