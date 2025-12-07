# 多数据源（JPA）- 完全学习指南

## 📚 简介

在企业应用中，经常需要同时操作多个数据库。Spring Boot 提供了灵活的方式来配置和管理多个数据源。这个 demo 展示了如何使用 JPA 实现多数据源支持。

### 为什么学习这个？
- 🎯 **数据库分离：读写分离、业务库隔离**
- 🎯 **多库支持：同时访问多个数据库**
- 🎯 **灵活切换：运行时动态切换数据源**
- 🎯 **高可用：主从切换、灾难恢复**

---

## 🎯 核心概念

### 1. 多数据源的常见场景

```
场景 1: 读写分离
┌──────────────┐
│  主数据库    │ ← 写操作
│  (Master)    │
└──────────────┘
       │
    ┌──┴──┐
    ↓     ↓
┌──────┐ ┌──────┐
│从1   │ │从2   │ ← 读操作
│(Slave)
│ │(Slave)│
└──────┘ └──────┘

场景 2: 业务库隔离
┌──────────┐ ┌──────────┐ ┌──────────┐
│用户库    │ │订单库    │ │日志库    │
│(userdb) │ │(orderdb) │ │(logdb)  │
└──────────┘ └──────────┘ └──────────┘

场景 3: 异地多活
┌──────────┐ ┌──────────┐
│数据中心1 │ │数据中心2 │
└──────────┘ └──────────┘
```

### 2. 配置多数据源

```java
// 配置文件
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/db1
      username: root
      password: root
      driver-class-name: com.mysql.cj.jdbc.Driver
    secondary:
      url: jdbc:mysql://localhost:3306/db2
      username: root
      password: root
      driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

### 3. 配置类实现多数据源

```java
@Configuration
public class DataSourceConfig {
    
    // 主数据源
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    // 从数据源
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    // 主数据源的 JPA 配置
    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            @Qualifier("primaryDataSource") DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean em = 
            new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.xkcoding.entity");
        
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(adapter);
        
        return em;
    }
    
    // 从数据源的 JPA 配置
    @Bean
    public LocalContainerEntityManagerFactoryBean secondaryEntityManagerFactory(
            @Qualifier("secondaryDataSource") DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean em = 
            new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.xkcoding.entity");
        
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(adapter);
        
        return em;
    }
    
    // 事务管理器
    @Bean
    @Primary
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManagerFactory") 
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }
    
    @Bean
    public PlatformTransactionManager secondaryTransactionManager(
            @Qualifier("secondaryEntityManagerFactory") 
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }
}
```

### 4. 使用多数据源

```java
// 主数据库的 Repository
@EnableJpaRepositories(
    basePackages = "com.xkcoding.repository.primary",
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryDataSourceConfig {
}

// 从数据库的 Repository
@EnableJpaRepositories(
    basePackages = "com.xkcoding.repository.secondary",
    entityManagerFactoryRef = "secondaryEntityManagerFactory",
    transactionManagerRef = "secondaryTransactionManager"
)
public class SecondaryDataSourceConfig {
}

// 主库 Repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}

// 从库 Repository
@Repository
public interface UserReadRepository extends JpaRepository<User, Long> {
}
```

### 5. 数据源动态切换

```java
@Component
public class DataSourceContextHolder {
    
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();
    
    public static void setDataSource(String dataSourceType) {
        CONTEXT_HOLDER.set(dataSourceType);
    }
    
    public static String getDataSource() {
        return CONTEXT_HOLDER.get() != null ? CONTEXT_HOLDER.get() : "primary";
    }
    
    public static void clearDataSource() {
        CONTEXT_HOLDER.remove();
    }
}

@Component
public class DynamicDataSource extends AbstractRoutingDataSource {
    
    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;
    
    @Autowired
    @Qualifier("secondaryDataSource")
    private DataSource secondaryDataSource;
    
    @PostConstruct
    public void init() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primaryDataSource);
        targetDataSources.put("secondary", secondaryDataSource);
        
        this.setTargetDataSources(targetDataSources);
        this.setDefaultTargetDataSource(primaryDataSource);
    }
    
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSource();
    }
}

// 使用
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUser(Long id) {
        // 从从数据库读取
        DataSourceContextHolder.setDataSource("secondary");
        try {
            return userRepository.findById(id).orElse(null);
        } finally {
            DataSourceContextHolder.clearDataSource();
        }
    }
    
    public User saveUser(User user) {
        // 写入主数据库
        DataSourceContextHolder.setDataSource("primary");
        try {
            return userRepository.save(user);
        } finally {
            DataSourceContextHolder.clearDataSource();
        }
    }
}
```

---

## 💡 实现细节

### 1. 读写分离的最佳实践

```java
@Service
public class ReadWriteSeparationService {
    
    @Autowired
    private UserRepository userRepository;
    
    // 只读操作使用从库
    @Transactional(readOnly = true)
    public User getUserInfo(Long userId) {
        DataSourceContextHolder.setDataSource("secondary");
        try {
            return userRepository.findById(userId).orElse(null);
        } finally {
            DataSourceContextHolder.clearDataSource();
        }
    }
    
    // 写操作使用主库
    @Transactional(readOnly = false)
    public User updateUser(User user) {
        DataSourceContextHolder.setDataSource("primary");
        try {
            return userRepository.save(user);
        } finally {
            DataSourceContextHolder.clearDataSource();
        }
    }
}
```

### 2. AOP 自动切换数据源

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface DataSource {
    String value() default "primary";
}

@Aspect
@Component
public class DataSourceAspect {
    
    @Before("@annotation(dataSource)")
    public void switchDataSource(JoinPoint jp, DataSource dataSource) {
        DataSourceContextHolder.setDataSource(dataSource.value());
    }
    
    @After("@annotation(dataSource)")
    public void restoreDataSource() {
        DataSourceContextHolder.clearDataSource();
    }
}

// 使用
@Service
public class UserService {
    
    @DataSource("secondary")  // 从从库读取
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    @DataSource("primary")  // 写入主库
    public User saveUser(User user) {
        return userRepository.save(user);
    }
}
```

---

## 🏢 行业应用

### 1. 读写分离

```
高流量应用，读操作远多于写操作
→ 所有读操作指向从库
→ 所有写操作指向主库
→ 主从同步
```

### 2. 多地域部署

```
支持多地域，减少网络延迟
→ 用户从最近的数据中心读取
→ 关键数据异步同步到其他中心
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/db1
      username: root
      password: root
    secondary:
      url: jdbc:mysql://localhost:3306/db2
      username: root
      password: root
```

---

## 🧪 测试

```java
@SpringBootTest
class MultiDataSourceTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    public void testReadFromSecondary() {
        User user = userService.getUser(1L);
        assertNotNull(user);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| DataSource | 数据源 |
| AbstractRoutingDataSource | 动态数据源路由 |
| ThreadLocal | 线程本地变量，存储数据源标识 |
| @EnableJpaRepositories | 启用 JPA Repository |
| 读写分离 | 主从同步 |
| 动态切换 | 运行时选择数据源 |

---

**恭喜！** 🎉 你已经掌握了多数据源的配置和使用！
