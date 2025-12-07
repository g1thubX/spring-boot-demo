# Spring Session 共享会话 - 完全学习指南

## 📚 简介

在分布式系统中，使用 Spring Session 可以实现会话共享，允许多个服务器实例访问同一个会话数据，解决分布式 Session 问题。

### 核心特点
- 🎯 **分布式支持：多个服务器实例共享 Session**
- 🎯 **持久化：Session 存储在 Redis、数据库等**
- 🎯 **应用透明：代码改动最少**
- 🎯 **灵活存储：支持多种存储方案**

---

## 🎯 使用方法

### 1. 添加依赖

```xml
<!-- Redis 存储 Session -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  session:
    store-type: redis  # 使用 Redis 存储
    redis:
      namespace: spring:session  # Redis 键前缀
  redis:
    host: localhost
    port: 6379
```

### 3. 使用 Session

```java
@RestController
public class UserController {
    
    @GetMapping("/login")
    public String login(HttpSession session) {
        // 存储到 Session（自动同步到 Redis）
        session.setAttribute("userId", 123L);
        session.setAttribute("username", "Alice");
        
        return "登录成功";
    }
    
    @GetMapping("/user-info")
    public Map<String, Object> getUserInfo(HttpSession session) {
        // 从 Session 读取（从 Redis 读取）
        Long userId = (Long) session.getAttribute("userId");
        String username = (String) session.getAttribute("username");
        
        return Map.of("userId", userId, "username", username);
    }
    
    @GetMapping("/logout")
    public String logout(HttpSession session) {
        // 销毁 Session
        session.invalidate();
        return "登出成功";
    }
}
```

### 4. 集成 Spring Security

```java
@Configuration
@EnableSpringHttpSession
public class SessionConfig {
    
    @Bean
    public SessionRepository sessionRepository() {
        return new RedisIndexedSessionRepository(null);
    }
}

@RestController
public class AuthController {
    
    @PostMapping("/login")
    public String login(String username, String password, HttpSession session) {
        // 验证用户
        User user = userService.authenticate(username, password);
        
        if (user != null) {
            // 存储认证信息
            session.setAttribute("user", user);
            session.setAttribute("authenticated", true);
            return "登录成功";
        }
        
        return "登录失败";
    }
}
```

---

## 💡 实现细节

### 1. 会话超时配置

```yaml
spring:
  session:
    timeout: 1800  # 会话超时时间（秒），默认 30 分钟
    redis:
      flush-mode: on_save  # 何时保存 Session
```

### 2. 自定义会话监听

```java
@Configuration
public class SessionEventConfig {
    
    @Bean
    public HttpSessionEventPublisher httpSessionEventPublisher() {
        return new HttpSessionEventPublisher();
    }
}

@Component
public class SessionListener implements HttpSessionListener {
    
    @Override
    public void sessionCreated(HttpSessionEvent event) {
        log.info("会话创建: " + event.getSession().getId());
    }
    
    @Override
    public void sessionDestroyed(HttpSessionEvent event) {
        log.info("会话销毁: " + event.getSession().getId());
    }
}
```

### 3. 多 Cookie 支持

```yaml
spring:
  session:
    servlet:
      cookie:
        name: CUSTOM_SESSION  # 自定义 Cookie 名
        domain: .example.com  # 跨域 Cookie
        path: /
        http-only: true
        secure: true  # HTTPS 时启用
        max-age: 1800
```

---

## 🏢 行业应用

### 分布式系统会话共享

```
用户登录 → 服务器 A（Session 存储到 Redis）
         ↓
用户请求 → 服务器 B（从 Redis 读取 Session）
         ↓
用户操作 → 任意服务器（都能读取同一 Session）
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Spring Session | Session 共享框架 |
| SessionRepository | Session 存储库 |
| Redis | Session 存储后端 |
| 会话超时 | Session 生存时间 |
| Cookie | 会话标识传输 |

---

**恭喜！** 🎉 你已经掌握了 Spring Session！
