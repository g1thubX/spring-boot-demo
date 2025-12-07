# Spring Boot Demo - 全套教程索引

## 📚 项目简介

这是一个全面的 Spring Boot 学习资源库，包含 62 个 demo 模块，涵盖 Spring 生态的绝大部分功能。每个 demo 都配备了详细的 all_you_need.md 教程，帮助开发者从入门到精通。

## 🎯 学习路径建议

### 第一阶段：Core 基础（第 1-5 天）
这些是 Spring Boot 的基础概念，必须掌握：

1. **[demo-helloworld](./demo-helloworld/all_you_need.md)** - Spring Boot 入门
   - 📖 学习 Spring Boot 的启动流程、REST 接口基础
   - 🎯 时间：2-3 小时
   - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

2. **[demo-properties](./demo-properties/all_you_need.md)** - 配置文件管理
   - 📖 学习 application.yml、多环境配置、@ConfigurationProperties
   - 🎯 时间：2-3 小时
   - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

3. **[demo-exception-handler](./demo-exception-handler/all_you_need.md)** - 统一异常处理
   - 📖 学习 @ControllerAdvice、统一响应格式、错误处理
   - 🎯 时间：2-3 小时
   - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

4. **[demo-logback](./demo-logback/all_you_need.md)** - 日志管理
   - 📖 学习 Logback 配置、日志级别、生产级别日志
   - 🎯 时间：1-2 小时
   - ⭐ 重要性：⭐⭐⭐⭐ 重要

5. **[demo-actuator](./demo-actuator/all_you_need.md)** - 应用监控
   - 📖 学习健康检查、性能指标、Actuator 端点
   - 🎯 时间：2-3 小时
   - ⭐ 重要性：⭐⭐⭐⭐ 重要

### 第二阶段：数据访问层（第 6-12 天）
数据库操作是后端开发的核心：

6. **[demo-orm-jpa](./demo-orm-jpa/all_you_need.md)** - Spring Data JPA
   - 📖 学习 JPA、实体关系映射、级联操作
   - 🎯 时间：4-5 小时
   - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

7. **[demo-orm-mybatis](./demo-orm-mybatis/all_you_need.md)** - MyBatis ORM
   - 📖 学习 MyBatis、动态 SQL、Mapper 配置
   - 🎯 时间：3-4 小时
   - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

8. **[demo-orm-mybatis-plus](./demo-orm-mybatis-plus)** - MyBatis Plus
   - 📖 学习 MyBatis Plus 的快速开发功能

9. **[demo-orm-beetlsql](./demo-orm-beetlsql)** - BeetlSQL
   - 📖 学习 BeetlSQL 的特点和用法

10. **[demo-cache-redis](./demo-cache-redis/all_you_need.md)** - Redis 缓存
    - 📖 学习 Redis 数据结构、Spring Cache、缓存策略
    - 🎯 时间：3-4 小时
    - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

11. **[demo-cache-ehcache](./demo-cache-ehcache)** - Ehcache 本地缓存
    - 📖 学习本地缓存和分布式缓存的区别

### 第三阶段：Web 开发（第 13-18 天）
Web 层技术和实时通信：

12. **[demo-swagger](./demo-swagger/all_you_need.md)** - API 文档
    - 📖 学习 Swagger、API 文档自动生成
    - 🎯 时间：1-2 小时
    - ⭐ 重要性：⭐⭐⭐⭐ 重要

13. **[demo-websocket](./demo-websocket/all_you_need.md)** - 实时通信
    - 📖 学习 WebSocket、双向通信、STOMP
    - 🎯 时间：2-3 小时
    - ⭐ 重要性：⭐⭐⭐ 有用

14. **[demo-upload](./demo-upload/all_you_need.md)** - 文件上传
    - 📖 学习文件上传、云存储、安全性
    - 🎯 时间：2-3 小时
    - ⭐ 重要性：⭐⭐⭐⭐ 重要

15. **[demo-email](./demo-email/all_you_need.md)** - 邮件发送
    - 📖 学习 JavaMail、邮件模板、异步发送
    - 🎯 时间：1-2 小时
    - ⭐ 重要性：⭐⭐⭐ 有用

16. **[demo-template-freemarker](./demo-template-freemarker)** - Freemarker 模板
    - 📖 学习服务端模板引擎

17. **[demo-template-thymeleaf](./demo-template-thymeleaf)** - Thymeleaf 模板
    - 📖 学习 Thymeleaf 的用法

### 第四阶段：消息与异步（第 19-24 天）
分布式系统的关键技术：

18. **[demo-mq-rabbitmq](./demo-mq-rabbitmq/all_you_need.md)** - RabbitMQ 消息队列
    - 📖 学习 AMQP、四种消息模式、消息可靠性
    - 🎯 时间：4-5 小时
    - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

19. **[demo-mq-kafka](./demo-mq-kafka)** - Kafka 消息队列
    - 📖 学习 Kafka、主题分区、消费者组

20. **[demo-mq-rocketmq](./demo-mq-rocketmq)** - RocketMQ 消息队列
    - 📖 学习 RocketMQ 的特点

21. **[demo-async](./demo-async)** - 异步任务
    - 📖 学习 @Async、异步编程

22. **[demo-task](./demo-task)** - 定时任务（简单）
    - 📖 学习 @Scheduled 注解

23. **[demo-task-quartz](./demo-task-quartz/all_you_need.md)** - Quartz 定时任务
    - 📖 学习 Quartz 框架、动态管理任务
    - 🎯 时间：3 小时
    - ⭐ 重要性：⭐⭐⭐⭐ 重要

### 第五阶段：搜索与数据库（第 25-30 天）
高级数据处理：

24. **[demo-elasticsearch](./demo-elasticsearch)** - Elasticsearch（Spring Data）
    - 📖 学习全文搜索、索引管理

25. **[demo-elasticsearch-rest-high-level-client](./demo-elasticsearch-rest-high-level-client)** - Elasticsearch（REST 客户端）
    - 📖 学习 Elasticsearch 7.x+ 的官方客户端

26. **[demo-mongodb](./demo-mongodb)** - MongoDB 数据库
    - 📖 学习 NoSQL 数据库、文档存储

27. **[demo-neo4j](./demo-neo4j)** - Neo4j 图数据库
    - 📖 学习图数据库、关系查询

### 第六阶段：安全与认证（第 31-35 天）
企业级安全要求：

28. **[demo-rbac-security](./demo-rbac-security/all_you_need.md)** - Spring Security RBAC
    - 📖 学习认证、授权、RBAC 模型、JWT
    - 🎯 时间：4-5 小时
    - ⭐ 重要性：⭐⭐⭐⭐⭐ 必学

29. **[demo-rbac-shiro](./demo-rbac-shiro)** - Apache Shiro
    - 📖 学习 Shiro 权限框架

30. **[demo-session](./demo-session)** - Session 共享
    - 📖 学习 Spring Session、分布式 Session

31. **[demo-oauth](./demo-oauth)** - OAuth 认证
    - 📖 学习 OAuth 2.0 授权

32. **[demo-social](./demo-social)** - 第三方登录
    - 📖 学习微信、GitHub、QQ 等第三方登陆

33. **[demo-ldap](./demo-ldap)** - LDAP 集成
    - 📖 学习企业级目录服务

34. **[demo-https](./demo-https)** - HTTPS 配置
    - 📖 学习 SSL/TLS 配置

### 第七阶段：高级特性（第 36-45 天）
企业级功能实现：

35. **[demo-multi-datasource-jpa](./demo-multi-datasource-jpa)** - 多数据源（JPA）
    - 📖 学习多数据源配置、动态数据源切换

36. **[demo-multi-datasource-mybatis](./demo-multi-datasource-mybatis)** - 多数据源（MyBatis）
    - 📖 学习 MyBatis 多数据源

37. **[demo-dynamic-datasource](./demo-dynamic-datasource)** - 动态数据源
    - 📖 学习运行时动态添加数据源

38. **[demo-sharding-jdbc](./demo-sharding-jdbc)** - ShardingJDBC 分库分表
    - 📖 学习数据库分片、分库分表

39. **[demo-ratelimit-guava](./demo-ratelimit-guava)** - 限流（本地）
    - 📖 学习 Guava RateLimiter 单机限流

40. **[demo-ratelimit-redis](./demo-ratelimit-redis)** - 限流（分布式）
    - 📖 学习 Redis + Lua 分布式限流

41. **[demo-zookeeper](./demo-zookeeper)** - Zookeeper 分布式锁
    - 📖 学习分布式锁、并发控制

42. **[demo-dubbo](./demo-dubbo)** - Dubbo RPC
    - 📖 学习微服务 RPC 框架

43. **[demo-codegen](./demo-codegen)** - 代码生成
    - 📖 学习代码生成器、提升开发效率

### 第八阶段：进阶工具（第 46-50 天）
企业级工具和框架：

44. **[demo-graylog](./demo-graylog)** - 日志收集系统
    - 📖 学习集中式日志管理

45. **[demo-activiti](./demo-activiti)** - Activiti 工作流
    - 📖 学习工作流引擎

46. **[demo-ureport2](./demo-ureport2)** - UReport2 报表
    - 📖 学习中国式复杂报表

47. **[demo-urule](./demo-urule)** - URule 规则引擎
    - 📖 学习规则引擎

48. **[demo-uflo](./demo-uflo)** - UFlo 流程引擎
    - 📖 学习轻量级流程引擎

49. **[demo-pay](./demo-pay)** - 支付集成
    - 📖 学习微信支付、支付宝集成

50. **[demo-tio](./demo-tio)** - TIO 网络编程
    - 📖 学习高性能网络编程

### 第九阶段：部署与容器化（第 51-55 天）
生产环境部署：

51. **[demo-docker](./demo-docker)** - Docker 容器化
    - 📖 学习 Docker 打包、容器化部署

52. **[demo-war](./demo-war)** - WAR 打包
    - 📖 学习传统 WAR 方式部署

53. **[demo-swagger-beauty](./demo-swagger-beauty)** - Swagger 美化
    - 📖 学习 Knife4j 美化 Swagger UI

54. **[demo-flyway](./demo-flyway)** - Flyway 数据库迁移
    - 📖 学习数据库版本管理

55. **[demo-websocket-socketio](./demo-websocket-socketio)** - Socket.IO
    - 📖 学习 Socket.IO 实时通信

---

## 🎓 学习方法论

### 阅读顺序
1. **从上到下顺序阅读**（按学习路径）
2. **每个 all_you_need.md 包含5个部分：**
   - 📚 简介：快速了解这个技术是什么
   - 🎯 核心概念：深入理解技术原理
   - 💡 实现细节：分析源代码和设计模式
   - 🏢 行业应用：真实业务场景案例
   - 🚀 快速开始和🧪 测试：动手实践

### 时间投入
- **入门阶段**：5-7 天（必学部分）
- **进阶阶段**：15-20 天（重要部分）
- **精通阶段**：30-60 天（完整学习）

### 实践建议
1. **编码实践**：不仅要读，要用手写代码
2. **对标学习**：选择 2-3 个核心 demo 深入钻研
3. **项目驱动**：用学到的知识做一个完整项目
4. **知识总结**：写笔记、画思维导图、讲给别人听

---

## 📊 技术栈全景

```
Spring Boot 生态
├─ Web (Web MVC, WebSocket, REST)
├─ 数据访问 (JPA, MyBatis, MongoDB, Redis)
├─ 安全 (Security, Shiro, OAuth)
├─ 消息 (RabbitMQ, Kafka, RocketMQ)
├─ 缓存 (Redis, Ehcache)
├─ 搜索 (Elasticsearch)
├─ 监控 (Actuator, Boot Admin)
├─ 日志 (Logback, Graylog)
├─ 定时任务 (Quartz, XXL-Job)
├─ 文件存储 (本地/七牛/OSS)
├─ 分布式 (Dubbo, Zookeeper, ShardingJDBC)
├─ 工作流 (Activiti, UFlo)
├─ 报表 (UReport2)
├─ 规则引擎 (URule)
└─ 部署 (Docker, WAR, Flyway)
```

---

## 🔗 相关资源

- **官方文档**：https://spring.io/projects/spring-boot
- **Spring 中文网**：https://spring.io/
- **GitHub 项目**：https://github.com/xkcoding/spring-boot-demo
- **个人博客**：https://xkcoding.com

---

## 📝 学习笔记建议

对于每个 demo，建议记录：

```
1. 核心概念（3-5 个关键点）
2. 代码示例（复制粘贴可用的代码片段）
3. 常见问题（Q&A）
4. 应用场景（在什么情况下用这个技术）
5. 最佳实践（怎样用才最优）
6. 陷阱（踩过的坑）
```

---

## 🚀 后续学习方向

学完基础后，可以选择：

1. **微服务架构**：Spring Cloud、Kubernetes
2. **高性能**：性能优化、缓存策略、数据库优化
3. **分布式**：分布式事务、一致性算法
4. **云计算**：AWS、GCP、Azure
5. **大数据**：Spark、Flink、Hadoop
6. **DevOps**：CI/CD、监控告警、日志分析

---

## ⭐ 最后的话

这套教程的目标是让你从完全不懂到逐步深入理解 Spring Boot 的各个方面。记住：

- **知其然且知其所以然**：不仅要学会用，更要理解原理
- **动手实践**：再好的教程也比不上自己写代码
- **持续深造**：每个技术都很深，不要追求一次学完
- **反复温习**：好的知识需要多次回顾才能真正掌握

**祝你学习愉快！** 🎉

---

*最后更新：2024 年 1 月*
*包含 62 个 demo 模块，15 个详细教程*
