# Spring Boot 配置文件 - 完全学习指南

## 📚 简介

在企业应用开发中，不同的环境（开发、测试、生产）需要不同的配置。Spring Boot 提供了强大而灵活的配置管理机制，让你能够轻松处理这些差异。

### 为什么学习这个？
- 🎯 **学会外部化配置，让应用更灵活**
- 🎯 **掌握多环境配置管理**
- 🎯 **了解配置优先级，避免配置冲突**
- 🎯 **为生产部署打下基础**

### 核心价值
- **环境隔离**：同一个应用在不同环境使用不同配置
- **安全性**：敏感信息（如密码）不需要硬编码在代码里
- **可维护性**：修改配置无需重新编译应用
- **灵活性**：运行时动态注入配置值

---

## 🎯 核心概念详解

### 1. Spring Boot 配置体系

Spring Boot 支持多种配置方式，优先级从高到低：

```
1. 命令行参数 (--key=value)
   ↓
2. 系统环境变量 (System.getenv)
   ↓
3. application-{profile}.properties/yml (激活的 profile)
   ↓
4. application.properties/yml (默认配置)
   ↓
5. @PropertySource 注解指定的配置文件
```

**示例：** 如果三个配置都设置了 `server.port`：
```
命令行: java -jar app.jar --server.port=9999
环境变量: export SERVER_PORT=8888
application-dev.yml: server.port: 8081
application.yml: server.port: 8080

最终生效的值是: 9999 (命令行最优先)
```

### 2. 配置文件格式：YAML vs Properties

**YAML 格式（推荐）：** application.yml
```yaml
server:                    # 层级结构，使用缩进
  port: 8080
  servlet:
    context-path: /demo

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db
    username: root
    password: root
```

**Properties 格式：** application.properties
```properties
# 使用点号分隔层级
server.port=8080
server.servlet.context-path=/demo

spring.datasource.url=jdbc:mysql://localhost:3306/db
spring.datasource.username=root
spring.datasource.password=root
```

**为什么选择 YAML？**
- 更易读，缩进清晰
- 支持复杂的数据结构
- 错误时有更好的提示

### 3. 三种注入配置的方法

#### 方法 1：@Value 注解（简单配置）

```java
@Component
public class ApplicationProperty {
    // 注入单个配置值
    @Value("${application.name}")
    private String name;
    
    @Value("${application.version}")
    private String version;
    
    // 提供默认值（配置不存在时使用）
    @Value("${application.description:Default Description}")
    private String description;
    
    // 注入系统属性
    @Value("${user.home}")
    private String userHome;
    
    // 注入环境变量
    @Value("${java.version}")
    private String javaVersion;
}
```

**优点：**
- 简单直接
- 适合注入单个值

**缺点：**
- 配置分散在代码各处
- 容易出现拼写错误

#### 方法 2：@ConfigurationProperties（推荐）

```java
@Data
@Component
@ConfigurationProperties(prefix = "developer")
public class DeveloperProperty {
    // 对应配置: developer.name
    private String name;
    
    // 对应配置: developer.website
    private String website;
    
    // 对应配置: developer.qq
    private String qq;
    
    // 对应配置: developer.phone-number (自动转换驼峰)
    private String phoneNumber;
    
    // 复杂对象
    private Map<String, String> contacts;
    
    // 列表
    private List<String> skills;
}
```

**配置文件（application.yml）：**
```yaml
developer:
  name: John
  website: https://example.com
  qq: 123456789
  phone-number: 13800138000
  contacts:
    github: https://github.com/john
    twitter: https://twitter.com/john
  skills:
    - Java
    - Spring Boot
    - Kubernetes
```

**优点：**
- 类型安全
- 支持复杂对象
- 配置集中在一个类中
- 支持自动验证（@Valid）
- IDE 有更好的代码提示

#### 方法 3：Environment 接口（程序式访问）

```java
@Component
public class ConfigAccessor {
    @Autowired
    private Environment environment;
    
    public void printConfigs() {
        // 读取配置
        String appName = environment.getProperty("application.name");
        String port = environment.getProperty("server.port");
        
        // 提供默认值
        String description = environment.getProperty(
            "application.description", 
            "Default Description"
        );
        
        // 读取为其他类型
        Integer portInt = environment.getProperty(
            "server.port", 
            Integer.class
        );
    }
}
```

### 4. 多环境配置

**文件结构：**
```
src/main/resources/
├── application.yml          (公共配置)
├── application-dev.yml      (开发环境)
├── application-test.yml     (测试环境)
└── application-prod.yml     (生产环境)
```

**application.yml（公共配置）：**
```yaml
spring:
  application:
    name: my-app
  jpa:
    show-sql: false

# 指定激活的默认 profile
  profiles:
    active: dev
```

**application-dev.yml（开发环境）：**
```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp_dev
    username: root
    password: root
  jpa:
    show-sql: true  # 开发环境显示SQL
```

**application-prod.yml（生产环境）：**
```yaml
server:
  port: 8888

spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/myapp
    username: prod_user
    password: ${DB_PASSWORD}  # 从环境变量读取，不硬编码
  jpa:
    show-sql: false  # 生产环境不显示SQL
```

**激活环境的几种方式：**

1. **application.yml 中指定：**
```yaml
spring:
  profiles:
    active: prod
```

2. **启动时指定：**
```bash
java -jar app.jar --spring.profiles.active=prod
```

3. **环境变量：**
```bash
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar
```

4. **IDE 中指定：**
在 IDE 的 Run Configuration 中设置 VM Options：
```
-Dspring.profiles.active=prod
```

### 5. 配置文件 IDE 支持

为了让 IDE 识别自定义配置，避免红线警告：

**pom.xml 添加：**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

**创建文件：** src/main/resources/META-INF/additional-spring-configuration-metadata.json
```json
{
  "properties": [
    {
      "name": "application.name",
      "type": "java.lang.String",
      "description": "应用名称"
    },
    {
      "name": "application.version",
      "type": "java.lang.String",
      "description": "应用版本"
    },
    {
      "name": "developer.name",
      "type": "java.lang.String",
      "description": "开发者名称"
    }
  ]
}
```

---

## 💡 实现细节深入分析

### 1. 自动类型转换

Spring Boot 会自动将字符串配置转换为对应的 Java 类型：

```yaml
# 配置文件
server:
  port: 8080                    # -> Integer
  shutdown: graceful            # -> Enum
  error:
    include-message: always     # -> Boolean

spring:
  jpa:
    show-sql: true              # -> Boolean
  datasource:
    hikari:
      maximum-pool-size: 20     # -> Integer
      connection-timeout: 30000  # -> Long
```

```java
// 对应的属性类型
@Data
@ConfigurationProperties(prefix = "server")
class ServerConfig {
    private Integer port;          // 自动转换为 Integer
    private Shutdown shutdown;     // 自动转换为 Enum
}
```

### 2. 驼峰命名自动转换

配置文件中的 kebab-case 会自动转换为 Java 中的 camelCase：

```yaml
developer:
  phone-number: 13800138000      # phone-number
  company-name: TechCorp         # company-name
  first-login-time: 2024-01-01   # first-login-time
```

```java
@Data
@ConfigurationProperties(prefix = "developer")
class DeveloperConfig {
    private String phoneNumber;        // 自动对应 phone-number
    private String companyName;        // 自动对应 company-name
    private Date firstLoginTime;       // 自动对应 first-login-time
}
```

### 3. 占位符和默认值

在配置文件中可以使用占位符：

```yaml
app:
  name: MyApp
  version: 1.0.0
  description: ${app.name} v${app.version}  # 引用其他配置
  
database:
  url: jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/myapp
  # 如果没有 DB_HOST 环境变量，使用默认值 localhost
  # 如果没有 DB_PORT 环境变量，使用默认值 3306
```

### 4. 环境变量映射

环境变量名称规则（都会被转换为小写并用点号分隔）：

```bash
# 环境变量设置
export SPRING_DATASOURCE_URL=jdbc:mysql://db.example.com/myapp
export SPRING_DATASOURCE_USERNAME=user
export APP_DEVELOPER_NAME=John

# 对应的配置键
spring.datasource.url
spring.datasource.username
app.developer.name
```

---

## 🏢 行业应用举例

### 1. 多数据库环境配置

```yaml
# application-dev.yml (开发)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp_dev
    username: root
    password: root
  jpa:
    hibernate:
      ddl-auto: create-drop  # 每次启动重建表

# application-prod.yml (生产)
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/myapp
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
  jpa:
    hibernate:
      ddl-auto: validate  # 只验证，不修改表结构
    show-sql: false
```

### 2. 功能开关配置

```yaml
# application.yml
features:
  enable-cache: true
  enable-async: true
  enable-scheduled-jobs: false
  new-ui-beta: false

logging:
  level:
    com.mycompany: ${LOG_LEVEL:INFO}
```

```java
@Component
@ConfigurationProperties(prefix = "features")
@Data
public class FeatureConfig {
    private Boolean enableCache;
    private Boolean enableAsync;
    private Boolean enableScheduledJobs;
    private Boolean newUiBeta;
}

@RestController
public class UserController {
    @Autowired
    private FeatureConfig featureConfig;
    
    @GetMapping("/users")
    public List<User> listUsers() {
        if (featureConfig.getEnableCache()) {
            // 使用缓存的实现
            return cachedUserService.getUsers();
        } else {
            // 不使用缓存
            return userService.getUsers();
        }
    }
}
```

### 3. 微服务配置中心

```yaml
# application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mycompany/config-repo

  application:
    name: user-service

server:
  port: ${PORT:8080}  # 支持从环境变量动态指定端口
```

```bash
# 部署时动态传入配置
docker run -e PORT=9000 -e APP_ENV=prod myapp:latest
```

### 4. 敏感信息管理

**绝不要在配置文件中硬编码敏感信息！**

```yaml
# ❌ 错误做法
spring:
  datasource:
    password: admin123

# ✅ 正确做法 1: 使用环境变量
spring:
  datasource:
    password: ${DB_PASSWORD}

# ✅ 正确做法 2: 使用 Spring Cloud Config Server
# ✅ 正确做法 3: 使用 Vault（HashiCorp）
# ✅ 正确做法 4: 使用云服务商的密钥管理服务
```

---

## 🚀 快速开始指南

### 1. 基本配置读取

**创建配置类：**
```java
@Data
@Component
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private String name;
    private String version;
    private String description;
}
```

**配置文件（application.yml）：**
```yaml
app:
  name: MyApplication
  version: 1.0.0
  description: A demo application
```

**在控制器中使用：**
```java
@RestController
public class ConfigController {
    @Autowired
    private AppConfig appConfig;
    
    @GetMapping("/config")
    public AppConfig getConfig() {
        return appConfig;
    }
}
```

### 2. 多环境启动

**创建配置文件：**
```
resources/
├── application.yml
├── application-dev.yml
├── application-test.yml
└── application-prod.yml
```

**启动命令：**
```bash
# 开发环境
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"

# 或者打包后
java -jar app.jar --spring.profiles.active=prod

# 或设置环境变量
export SPRING_PROFILES_ACTIVE=test
java -jar app.jar
```

### 3. 动态配置刷新

```java
@Component
public class DynamicConfigService {
    @Autowired
    private Environment environment;
    
    public void refreshConfig() {
        String newValue = environment.getProperty("app.name");
        log.info("刷新配置: {}", newValue);
    }
}
```

---

## 🧪 详细测试指南

### 测试 1：配置是否被正确读取

```bash
# 启动应用
mvn spring-boot:run

# 访问配置接口
curl http://localhost:8080/config

# 检查响应中的值是否正确
```

### 测试 2：@Value 注入

```java
@RestController
public class PropertyTestController {
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.version}")
    private String appVersion;
    
    @GetMapping("/test-value")
    public Map<String, String> testValue() {
        return Map.of(
            "appName", appName,
            "appVersion", appVersion
        );
    }
}
```

### 测试 3：@ConfigurationProperties 验证

```java
@Test
public void testConfigurationProperties() {
    assertThat(developerProperty.getName()).isNotEmpty();
    assertThat(developerProperty.getWebsite()).contains("http");
    assertThat(developerProperty.getPhoneNumber()).hasSize(11);
}
```

### 测试 4：多环境配置

```bash
# 开发环境测试
java -jar target/app.jar --spring.profiles.active=dev
# 查看日志确认加载的是 application-dev.yml

# 生产环境测试
java -jar target/app.jar --spring.profiles.active=prod
# 查看日志确认加载的是 application-prod.yml
```

### 测试 5：环境变量覆盖

```bash
# 设置环境变量（覆盖配置文件）
export SPRING_DATASOURCE_PASSWORD=new_password

# 启动应用
java -jar app.jar

# 应该使用新的密码而不是配置文件中的
```

---

## 🛠️ 开发实践指南

### 实践 1：创建分层配置类

需求：配置数据库、Redis、业务相关参数

```java
// 1. 数据库配置
@Data
@Component
@ConfigurationProperties(prefix = "db")
public class DatabaseConfig {
    private String host;
    private Integer port;
    private String database;
    private String username;
    private String password;
    private Integer poolSize;
}

// 2. Redis 配置
@Data
@Component
@ConfigurationProperties(prefix = "cache.redis")
public class RedisConfig {
    private String host;
    private Integer port;
    private String password;
    private Integer ttl;  // 超时时间，单位秒
}

// 3. 业务配置
@Data
@Component
@ConfigurationProperties(prefix = "business")
public class BusinessConfig {
    private Integer maxRetries;
    private Long requestTimeout;
    private Boolean enableNotification;
}
```

**配置文件（application.yml）：**
```yaml
db:
  host: localhost
  port: 3306
  database: myapp
  username: root
  password: root
  poolSize: 20

cache:
  redis:
    host: localhost
    port: 6379
    password: redis123
    ttl: 3600

business:
  max-retries: 3
  request-timeout: 5000
  enable-notification: true
```

### 实践 2：配置验证

需求：确保配置值的有效性

```java
@Data
@Component
@ConfigurationProperties(prefix = "db")
@Validated  // 启用验证
public class DatabaseConfig {
    @NotBlank(message = "数据库主机不能为空")
    private String host;
    
    @NotNull(message = "数据库端口不能为空")
    @Range(min = 1, max = 65535, message = "端口号范围是1-65535")
    private Integer port;
    
    @NotBlank(message = "数据库名不能为空")
    private String database;
    
    @Min(value = 1, message = "连接池大小至少为1")
    @Max(value = 100, message = "连接池大小不超过100")
    private Integer poolSize;
}
```

### 实践 3：条件化配置

需求：根据不同环境使用不同的服务实现

```java
@Configuration
public class ServiceConfiguration {
    
    @Bean
    @Profile("dev")
    public EmailService emailServiceForDev() {
        // 开发环境，打印日志而不真正发送邮件
        return new LoggingEmailService();
    }
    
    @Bean
    @Profile("prod")
    public EmailService emailServiceForProd() {
        // 生产环境，真正发送邮件
        return new SmtpEmailService();
    }
}
```

### 实践 4：配置的注入到 Bean

需求：将配置注入到自定义的 Service

```java
@Service
public class PaymentService {
    private final PaymentConfig config;
    
    // 构造器注入（推荐）
    public PaymentService(PaymentConfig config) {
        this.config = config;
    }
    
    public void processPayment(Order order) {
        // 使用配置
        String apiKey = config.getApiKey();
        String apiSecret = config.getApiSecret();
        Integer timeout = config.getTimeout();
        
        // 业务逻辑...
    }
}

@Data
@Component
@ConfigurationProperties(prefix = "payment")
class PaymentConfig {
    private String apiKey;
    private String apiSecret;
    private Integer timeout;
}
```

### 实践 5：配置热更新（结合 Spring Cloud）

在使用 Spring Cloud 时，可以实现配置的热更新：

```java
@RestController
@RefreshScope  // 配置变更时刷新此组件
public class FeatureToggleController {
    @Value("${features.new-ui-enabled}")
    private Boolean newUiEnabled;
    
    @GetMapping("/features/new-ui")
    public Boolean isNewUiEnabled() {
        // 当配置中心的值改变时，会自动刷新
        return newUiEnabled;
    }
}
```

---

## 📊 常见问题解决

### Q1: 配置文件中的密码安全吗？
**A:** 不安全。敏感信息应该：
1. 存储在环境变量中
2. 使用专门的密钥管理系统（如 Vault）
3. 在部署时注入

### Q2: 如何在配置中使用系统环境变量？
```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}  # 引用名为 DB_PASSWORD 的环境变量
    url: jdbc:mysql://${DB_HOST:localhost}:3306/myapp  # 有默认值
```

### Q3: application.yml 还是 application.properties？
**A:** 优先选择 YAML，因为：
- 更易读
- 支持复杂结构
- 现代 Spring Boot 推荐

### Q4: @Value 和 @ConfigurationProperties 的区别？
| 特性 | @Value | @ConfigurationProperties |
|-----|--------|------------------------|
| 使用场景 | 注入单个值 | 注入配置对象 |
| 类型安全 | 否 | 是 |
| 支持集合 | 否 | 是 |
| IDE 提示 | 无 | 有 |
| 验证支持 | 否 | 是 |

---

## 📖 扩展学习路径

1. **Spring Cloud Config** → 分布式配置管理
2. **Spring Cloud Consul/Nacos** → 配置中心和注册中心
3. **HashiCorp Vault** → 敏感信息管理
4. **Apollo** → 携程开源的配置中心
5. **Archaius** → Netflix 的配置框架

---

## 🎓 关键知识点总结

| 知识点 | 关键要点 |
|------|--------|
| 配置优先级 | 命令行 > 环境变量 > application-{profile}.yml > application.yml |
| YAML 格式 | 层级结构，缩进很重要 |
| @Value | 简单值注入，支持 SpEL |
| @ConfigurationProperties | 对象绑定，类型安全 |
| 多环境配置 | 通过 spring.profiles.active 指定 |
| 类型转换 | 自动进行，支持常见类型 |
| 占位符 | ${key:defaultValue} |
| 自定义配置验证 | 使用 @Validated 和 JSR-303 注解 |

---

**恭喜！** 🎉 你已经掌握了 Spring Boot 配置管理的精髓。良好的配置管理是构建可维护、可扩展应用的基础。
