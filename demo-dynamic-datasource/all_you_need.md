# 动态数据源 - 完全学习指南

## 📚 简介

动态数据源允许在应用运行时动态添加、删除、切换数据源，无需重启应用。这对于需要支持多租户、动态库表等场景非常重要。

### 为什么学习这个？
- 🎯 **多租户支持：每个租户使用独立数据库**
- 🎯 **数据隔离：更好的安全性和隐私**
- 🎯 **灵活扩展：无需重启应用即可添加数据源**
- 🎯 **成本优化：按需创建和销毁数据源**

---

## 🎯 核心概念

### 1. 动态切换数据源

```java
// 定义数据源标识常量
public class DataSourceNames {
    public static final String MASTER = "master";
    public static final String SLAVE = "slave";
}

// 数据源上下文
public class DataSourceContext {
    private static final ThreadLocal<String> CONTEXT = new ThreadLocal<>();
    
    public static void set(String dataSourceName) {
        CONTEXT.set(dataSourceName);
    }
    
    public static String get() {
        String result = CONTEXT.get();
        return result == null ? DataSourceNames.MASTER : result;
    }
    
    public static void clear() {
        CONTEXT.remove();
    }
}

// 动态数据源
@Component
public class DynamicDataSource extends AbstractRoutingDataSource {
    
    private final Map<Object, Object> dataSourceMap = new ConcurrentHashMap<>();
    
    @PostConstruct
    public void init() {
        // 设置默认数据源
        setDefaultTargetDataSource(createDataSource(DataSourceNames.MASTER));
    }
    
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContext.get();
    }
    
    // 动态添加数据源
    public void addDataSource(String dataSourceName, String url, 
                              String username, String password) {
        DataSource dataSource = createDataSource(url, username, password);
        dataSourceMap.put(dataSourceName, dataSource);
        setTargetDataSources(dataSourceMap);
        afterPropertiesSet();  // 刷新
    }
    
    // 删除数据源
    public void removeDataSource(String dataSourceName) {
        dataSourceMap.remove(dataSourceName);
        setTargetDataSources(dataSourceMap);
        afterPropertiesSet();
    }
}

// 使用
@Service
public class UserService {
    
    public User getUser(Long id) {
        DataSourceContext.set(DataSourceNames.MASTER);  // 使用主库
        try {
            return userRepository.findById(id).orElse(null);
        } finally {
            DataSourceContext.clear();
        }
    }
}
```

### 2. 多租户实现

```java
// 租户信息
@Data
public class TenantInfo {
    private Long tenantId;
    private String dataSourceName;
    private String url;
    private String username;
    private String password;
}

// 租户上下文
public class TenantContext {
    private static final ThreadLocal<TenantInfo> CONTEXT = new ThreadLocal<>();
    
    public static void set(TenantInfo tenantInfo) {
        CONTEXT.set(tenantInfo);
    }
    
    public static TenantInfo get() {
        return CONTEXT.get();
    }
}

// 拦截器：从 HTTP 请求中提取租户信息
@Component
public class TenantInterceptor implements HandlerInterceptor {
    
    @Autowired
    private TenantRepository tenantRepository;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        // 从请求头或 URL 中获取租户 ID
        String tenantIdStr = request.getHeader("X-Tenant-ID");
        Long tenantId = Long.parseLong(tenantIdStr);
        
        // 获取租户信息
        TenantInfo tenantInfo = tenantRepository.findById(tenantId).orElse(null);
        
        if (tenantInfo != null) {
            TenantContext.set(tenantInfo);
            DataSourceContext.set(tenantInfo.getDataSourceName());
        }
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, Exception ex) {
        TenantContext.clear();
        DataSourceContext.clear();
    }
}
```

### 3. AOP 方式切换数据源

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface DataSource {
    String value();
}

@Aspect
@Component
public class DataSourceAspect {
    
    @Before("@annotation(dataSource)")
    public void switchDataSource(JoinPoint jp, DataSource dataSource) {
        String dataSourceName = dataSource.value();
        DataSourceContext.set(dataSourceName);
    }
    
    @After("@annotation(dataSource)")
    public void restoreDataSource() {
        DataSourceContext.clear();
    }
}

// 使用
@Service
public class UserService {
    
    @DataSource("master")
    public User saveUser(User user) {
        return userRepository.save(user);
    }
    
    @DataSource("slave")
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
}
```

---

## 💡 实现细节

### 1. 初始化多个数据源

```yaml
# application.yml
datasources:
  master:
    url: jdbc:mysql://master:3306/mydb
    username: root
    password: root
    driver: com.mysql.cj.jdbc.Driver
  
  slave:
    url: jdbc:mysql://slave:3306/mydb
    username: root
    password: root
    driver: com.mysql.cj.jdbc.Driver
```

```java
@Configuration
public class DataSourceConfiguration {
    
    @Bean
    public DynamicDataSource dynamicDataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        
        // 从配置文件加载数据源
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("master", masterDataSource());
        dataSourceMap.put("slave", slaveDataSource());
        
        dynamicDataSource.setTargetDataSources(dataSourceMap);
        return dynamicDataSource;
    }
    
    @Bean
    public DataSource masterDataSource() {
        // 创建 master 数据源
        return DataSourceBuilder.create()
            .url("jdbc:mysql://localhost:3306/mydb")
            .username("root")
            .password("root")
            .build();
    }
    
    @Bean
    public DataSource slaveDataSource() {
        // 创建 slave 数据源
        return DataSourceBuilder.create()
            .url("jdbc:mysql://slave:3306/mydb")
            .username("root")
            .password("root")
            .build();
    }
}
```

### 2. 线程安全的数据源管理

```java
public class DataSourceManager {
    
    private final Map<String, DataSource> dataSourceMap = new ConcurrentHashMap<>();
    
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public void addDataSource(String name, DataSource dataSource) {
        lock.writeLock().lock();
        try {
            dataSourceMap.put(name, dataSource);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public DataSource getDataSource(String name) {
        lock.readLock().lock();
        try {
            return dataSourceMap.get(name);
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

---

## 🏢 行业应用

### 1. SaaS 多租户应用

```
用户A → 租户ID:1 → 数据库A
用户B → 租户ID:2 → 数据库B
用户C → 租户ID:3 → 数据库C
```

### 2. 分库分表

```
用户请求 → 确定 shard ID → 选择对应数据源 → 查询
```

---

## 🚀 快速开始

### 1. 配置

继承 `AbstractRoutingDataSource`，实现 `determineCurrentLookupKey()` 方法

### 2. 切换数据源

```java
DataSourceContext.set("dataSourceName");
// 执行数据库操作
DataSourceContext.clear();
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| AbstractRoutingDataSource | 动态数据源基类 |
| ThreadLocal | 线程本地变量 |
| 多租户 | 每个租户独立数据库 |
| 数据源切换 | 运行时动态选择 |
| 并发安全 | ConcurrentHashMap, ReadWriteLock |

---

**恭喜！** 🎉 你已经掌握了动态数据源的使用！
