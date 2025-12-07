# Logback 日志框架 - 完全学习指南

## 📚 简介

Logback 是 Java 现代日志框架，拥有更好的性能和更灵活的配置。它是 SLF4J 的原生实现，Spring Boot 默认使用 Logback。

### 为什么学习这个？
- 🎯 **理解日志级别和输出**
- 🎯 **掌握 Logback 配置**
- 🎯 **学会日志的最佳实践**
- 🎯 **处理生产环境的日志需求**

---

## 🎯 核心概念

### 1. 日志级别（从低到高）

```
TRACE   - 最详细的信息，一般不会启用
DEBUG   - 调试信息
INFO    - 一般信息，用于记录重要操作
WARN    - 警告信息，表示可能有问题
ERROR   - 错误信息，表示出现错误
FATAL   - 致命错误
```

### 2. Logback 三个主要组件

```
Logger        - 日志记录器，应用中的入口
├─ Appender   - 日志输出目标（控制台、文件等）
└─ Formatter  - 日志格式化
```

### 3. 配置文件：logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 定义变量 -->
    <property name="LOG_FILE" value="logs/app.log"/>
    <property name="LOG_PATTERN" value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"/>
    
    <!-- 控制台 Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>
    
    <!-- 文件 Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
        <!-- 每天滚动一次 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>10MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!-- 保留 30 天的日志 -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- Logger 配置 -->
    <logger name="com.xkcoding" level="DEBUG"/>
    <logger name="org.springframework" level="INFO"/>
    <logger name="org.hibernate" level="INFO"/>
    
    <!-- 根 Logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### 4. 日志使用最佳实践

```java
@Slf4j  // Lombok 提供的日志注解
@Service
public class UserService {
    
    public void createUser(User user) {
        // 使用参数化方式，避免字符串拼接
        log.debug("开始创建用户: {}", user.getUsername());
        
        try {
            userRepository.save(user);
            log.info("用户创建成功: {}", user.getId());
        } catch (Exception e) {
            log.error("创建用户失败: {}", user.getUsername(), e);
            throw new BusinessException("创建用户失败");
        }
    }
    
    // 不同日志级别的用途
    public void demonstrateLogLevels() {
        log.trace("追踪信息");  // 太详细，通常关闭
        log.debug("调试信息");  // 开发时有用
        log.info("普通信息");   // 记录重要事件
        log.warn("警告信息");   // 可能的问题
        log.error("错误信息");  // 需要注意的错误
    }
}
```

---

## 💡 实现细节

### 1. 日志输出格式说明

```
%d{HH:mm:ss.SSS}     - 时间戳
[%thread]            - 线程名
%-5level             - 日志级别（左对齐，5个字符）
%logger{36}          - Logger 名称（最多 36 个字符）
%msg                 - 日志信息
%n                   - 换行符
%X{traceId}          - MDC 变量（用于链路追踪）
%F:%L                - 文件名和行号
```

### 2. 日志分级输出

```xml
<!-- 只记录错误日志 -->
<appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>ERROR</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
    </filter>
    <file>logs/error.log</file>
    <encoder>
        <pattern>${LOG_PATTERN}</pattern>
    </encoder>
</appender>
```

### 3. 异步日志（提升性能）

```xml
<!-- 异步 Appender -->
<appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
    <queueSize>512</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <includeCallerData>false</includeCallerData>
</appender>

<root level="INFO">
    <appender-ref ref="ASYNC_FILE"/>
</root>
```

---

## 🏢 行业应用

### 1. 生产环境配置

```xml
<!-- application-prod.yml -->
logging:
  level:
    root: WARN
    com.xkcoding: INFO
  file: /var/log/application/app.log
  pattern:
    file: '%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n'
```

### 2. 链路追踪日志

```java
@Component
public class RequestIdInterceptor extends HandlerInterceptorAdapter {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        MDC.put("traceId", traceId);
        return true;
    }
}

// 日志格式: %X{traceId}
// 这样可以追踪整个请求链路
```

---

## 🚀 快速开始

### 1. 依赖（Spring Boot 默认包含）

```xml
<!-- Spring Boot 默认已包含 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

### 2. 创建 logback-spring.xml

在 `src/main/resources` 目录下创建 `logback-spring.xml`，Spring Boot 会自动加载。

### 3. 在代码中使用

```java
@Slf4j
@Service
public class ExampleService {
    public void example() {
        log.info("这是一条日志");
    }
}
```

---

## 🧪 测试

```bash
# 启动应用，查看控制台日志
mvn spring-boot:run

# 查看日志文件
tail -f logs/app.log
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| SLF4J | 日志门面，定义日志接口 |
| Logback | SLF4J 的实现，性能最好 |
| 日志级别 | TRACE < DEBUG < INFO < WARN < ERROR |
| Appender | 日志输出目标（控制台、文件等）|
| RollingPolicy | 日志文件滚动策略 |
| Pattern | 日志格式化模式 |
| MDC | Mapped Diagnostic Context，用于链路追踪 |

---

**恭喜！** 🎉 你已经掌握了 Logback 日志框架！
