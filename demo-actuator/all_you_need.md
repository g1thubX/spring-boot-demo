# Spring Boot Actuator 监控 - 完全学习指南

## 📚 简介

Spring Boot Actuator 提供了生产级别的应用监控和管理功能。它暴露了 REST 端点，可以查看应用的健康状态、性能指标、线程信息等，是微服务治理的重要工具。

### 为什么学习这个？
- 🎯 **掌握应用监控的核心概念**
- 🎯 **学会使用 Actuator 端点获取应用状态**
- 🎯 **集成与监控系统（如 Prometheus、Grafana）**
- 🎯 **理解应用健康检查和指标收集**

---

## 🎯 核心概念

### 1. Actuator 主要功能

```
监控 (Monitoring)
├─ 健康检查 (/actuator/health)
├─ 性能指标 (/actuator/metrics)
├─ 线程信息 (/actuator/threaddump)
└─ 堆内存 (/actuator/heapdump)

管理 (Management)
├─ 日志级别 (/actuator/loggers)
├─ 环境变量 (/actuator/env)
├─ Bean 信息 (/actuator/beans)
└─ 配置属性 (/actuator/configprops)

追踪 (Tracing)
├─ HTTP 追踪 (/actuator/httptrace)
├─ 审计事件 (/actuator/auditevents)
└─ 映射信息 (/actuator/mappings)
```

### 2. 配置和启用端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # 只暴露这些端点
        exclude: shutdown             # 排除这个端点
  endpoint:
    health:
      show-details: always            # 详细的健康信息
      show-components: always         # 显示健康检查组件
    shutdown:
      enabled: false                  # 禁用关闭端点
  metrics:
    enable:
      jvm: true
      process: true
      logback: true
```

### 3. 常用端点详解

#### a) 健康检查

```bash
# 查看应用健康状态
curl http://localhost:8080/actuator/health

# 响应
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000,
        "free": 200000000,
        "available": 150000000
      }
    }
  }
}
```

#### b) 应用信息

```bash
curl http://localhost:8080/actuator/info

# 响应
{
  "app": {
    "name": "My App",
    "version": "1.0.0"
  }
}
```

#### c) 性能指标

```bash
# 查看所有指标
curl http://localhost:8080/actuator/metrics

# 查看特定指标（如内存使用）
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# 响应
{
  "name": "jvm.memory.used",
  "description": "The amount of used memory",
  "baseUnit": "bytes",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 500000000
    }
  ],
  "availableTags": [...]
}
```

#### d) 日志管理

```bash
# 查看所有日志器
curl http://localhost:8080/actuator/loggers

# 修改日志级别（从 INFO 改为 DEBUG）
curl -X POST http://localhost:8080/actuator/loggers/com.xkcoding \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

#### e) 环境信息

```bash
# 查看所有环境变量和配置
curl http://localhost:8080/actuator/env

# 查看特定属性
curl http://localhost:8080/actuator/env/server.port
```

### 4. 自定义健康检查

```java
@Component
public class CustomHealthIndicator extends AbstractHealthIndicator {
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            // 检查自定义的服务或资源
            boolean isServiceHealthy = checkExternalService();
            
            if (isServiceHealthy) {
                builder.up()
                    .withDetail("service", "External service is reachable")
                    .withDetail("response-time", "50ms");
            } else {
                builder.down()
                    .withDetail("service", "External service is not reachable");
            }
        } catch (Exception e) {
            builder.unknown()
                .withException(e);
        }
    }
    
    private boolean checkExternalService() {
        // 检查外部服务的逻辑
        return true;
    }
}
```

### 5. 自定义指标

```java
@Service
public class CustomMetricsService {
    
    private final MeterRegistry meterRegistry;
    
    public CustomMetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordUserLogin(String username) {
        // 计数器：记录登录次数
        Counter.builder("user.login")
            .tag("username", username)
            .register(meterRegistry)
            .increment();
    }
    
    public void recordPaymentAmount(BigDecimal amount) {
        // 仪表：记录支付金额
        Gauge.builder("payment.amount", () -> amount.doubleValue())
            .register(meterRegistry);
    }
    
    public void recordRequestDuration(String endpoint, long duration) {
        // 计时器：记录请求耗时
        Timer.builder("http.request.duration")
            .tag("endpoint", endpoint)
            .register(meterRegistry)
            .record(Duration.ofMillis(duration));
    }
}
```

---

## 💡 实现细节

### 1. Actuator 的安全性

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                // 只有 ADMIN 角色才能访问 actuator 端点
                .antMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().permitAll()
            .and()
                .httpBasic();
    }
}
```

### 2. 导出指标到 Prometheus

```yaml
# pom.xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
```

然后访问 `http://localhost:8080/actuator/prometheus` 获取 Prometheus 格式的指标。

### 3. 与 Spring Boot Admin 集成

```yaml
# client: pom.xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>

# application.yml
spring:
  boot:
    admin:
      client:
        url: http://admin-server:8080
```

这样，应用会自动注册到 Boot Admin 服务器，可以在图形化界面查看所有应用的状态。

---

## 🏢 行业应用

### 1. 容器编排（Kubernetes）

健康检查端点可以用于 Kubernetes 的 readiness probe 和 liveness probe。

### 2. 自动报警

集成 Prometheus + Alertmanager，当指标超过阈值时自动报警。

### 3. 性能分析

收集各种指标，分析应用性能瓶颈。

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 2. 配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

### 3. 访问端点

```bash
curl http://localhost:8080/actuator/health
```

---

## 🧪 测试

```java
@SpringBootTest
class ActuatorTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    public void testHealthCheck() {
        ResponseEntity<String> response = restTemplate
            .getForEntity("/actuator/health", String.class);
        
        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertTrue(response.getBody().contains("UP"));
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Actuator | 应用监控工具 |
| Endpoint | 监控端点 |
| HealthIndicator | 健康检查指标 |
| Metrics | 性能指标收集 |
| MeterRegistry | 指标注册表 |
| Prometheus | 指标收集和存储 |

---

**恭喜！** 🎉 你已经掌握了 Spring Boot 的监控和管理！
