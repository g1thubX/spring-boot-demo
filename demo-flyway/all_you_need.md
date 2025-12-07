# Flyway 数据库迁移 - 完全学习指南

## 📚 简介

Flyway 是一个数据库迁移工具，自动管理数据库 schema 版本控制。应用启动时自动执行待迁移的 SQL 脚本，保证数据库结构与应用代码同步。

### 核心特点
- 🎯 **自动化：应用启动自动执行迁移**
- 🎯 **版本控制：追踪所有数据库变更**
- 🎯 **多环境：支持开发、测试、生产环境**

---

## 🎯 使用方法

### 1. 添加依赖

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration  # SQL 脚本位置
    baseline-on-migrate: true  # 基线版本
```

### 3. 创建迁移脚本

```
src/main/resources/db/migration/
├── V1__create_user_table.sql
├── V2__add_email_unique.sql
├── V3__create_order_table.sql
└── V4__add_indexes.sql
```

### 4. 脚本示例

```sql
-- V1__create_user_table.sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_email_unique.sql
ALTER TABLE user ADD UNIQUE KEY uk_email (email);

-- V3__create_order_table.sql
CREATE TABLE `order` (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user (id)
);

-- V4__add_indexes.sql
CREATE INDEX idx_user_email ON user (email);
CREATE INDEX idx_order_user_id ON `order` (user_id);
```

---

## 💡 实现细节

### 1. 回调脚本

```
src/main/resources/db/migration/
├── V1__schema.sql
├── V2__data.sql
├── U2__undo_data.sql  # 回滚脚本
└── R__views.sql  # 可重复脚本
```

### 2. 条件迁移

```java
@Configuration
public class FlywayConfig {
    
    @Bean
    public Flyway flyway(DataSource dataSource, @Value("${environment:dev}") String env) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/" + env)  // 不同环境不同脚本
            .load();
    }
}
```

### 3. 校验迁移

```yaml
spring:
  flyway:
    validate-on-migrate: true  # 验证已执行的脚本
    clean-disabled: false  # 允许清空数据库（开发环境）
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| V 脚本 | 版本化脚本 |
| U 脚本 | 撤销脚本 |
| R 脚本 | 可重复脚本 |
| flyway_schema_history | 迁移历史表 |
| baseline | 基线版本 |

---

**恭喜！** 🎉 你已经掌握了 Flyway 数据库迁移！
