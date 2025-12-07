# MyBatis Plus - 完全学习指南

## 📚 简介

MyBatis Plus 是 MyBatis 的增强工具，在 MyBatis 基础上提供了更强大的功能：自动生成 CRUD 代码、分页插件、性能分析、代码生成等。它让开发变得更快速，减少重复代码。

### 为什么学习这个？
- 🎯 **快速开发：自动生成 BaseMapper，减少 80% 的 CRUD 代码**
- 🎯 **性能优化：自带分页插件、性能分析工具**
- 🎯 **灵活查询：链式 QueryWrapper，条件查询更简洁**
- 🎯 **扩展性强：插件化架构，支持自定义功能**

---

## 🎯 核心概念

### 1. MyBatis Plus vs 原生 MyBatis

| 功能 | 原生 MyBatis | MyBatis Plus |
|-----|-----------|------------|
| CRUD | 手写 SQL | 自动生成 |
| 分页 | 手动计算 | 自动分页 |
| 条件查询 | 手写动态 SQL | QueryWrapper |
| 性能分析 | 无 | 内置 |
| 代码生成 | 无 | 自动生成 |
| 学习成本 | 高 | 低 |

### 2. 三个核心概念

```java
// 1. BaseMapper - 基础 CRUD 接口
@Mapper
public interface UserMapper extends BaseMapper<User> {
    // 自动获得：
    // insert(User) - 插入
    // deleteById(Long) - 删除
    // updateById(User) - 更新
    // selectById(Long) - 查询单个
    // selectList(QueryWrapper) - 条件查询
    // selectPage(Page, QueryWrapper) - 分页查询
}

// 2. QueryWrapper - 条件查询
QueryWrapper<User> qw = new QueryWrapper<>();
qw.eq("username", "alice")
  .ge("age", 18)
  .orderByDesc("id");
List<User> users = userMapper.selectList(qw);

// 3. IService - Service 层快速开发
public interface UserService extends IService<User> {
    // 自动获得：
    // save(User) - 保存
    // remove(Wrapper) - 删除
    // update(User, Wrapper) - 更新
    // getById(Long) - 查询单个
    // list(Wrapper) - 条件查询
    // page(Page, Wrapper) - 分页查询
}
```

### 3. QueryWrapper 链式查询

```java
// 基本查询条件
QueryWrapper<User> qw = new QueryWrapper<>();
qw.eq("status", 1);           // 等于
qw.ne("deleted", 1);           // 不等于
qw.gt("age", 18);              // 大于
qw.ge("age", 18);              // 大于等于
qw.lt("age", 65);              // 小于
qw.le("age", 65);              // 小于等于

// 模糊查询
qw.like("name", "alice");       // LIKE %alice%
qw.likeLeft("name", "alice");   // LIKE %alice
qw.likeRight("name", "alice");  // LIKE alice%

// 范围查询
qw.between("age", 18, 65);      // BETWEEN
qw.in("status", 1, 2, 3);       // IN
qw.notIn("status", 0);          // NOT IN

// 排序和分页
qw.orderByDesc("id");           // 倒序
qw.orderByAsc("create_time");   // 正序
qw.last("LIMIT 1");             // 最后执行的SQL片段

List<User> users = userMapper.selectList(qw);
```

### 4. LambdaQueryWrapper（类型安全）

```java
// 避免字符串写错，编译时检查
LambdaQueryWrapper<User> qw = new LambdaQueryWrapper<>();
qw.eq(User::getUsername, "alice")
  .ge(User::getAge, 18)
  .orderByDesc(User::getId);

List<User> users = userMapper.selectList(qw);
```

### 5. 分页查询

```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    public IPage<User> pageUsers(Integer pageNo, Integer pageSize) {
        Page<User> page = new Page<>(pageNo, pageSize);
        
        QueryWrapper<User> qw = new QueryWrapper<>();
        qw.eq("status", 1);
        
        IPage<User> result = userMapper.selectPage(page, qw);
        
        log.info("总数: {}, 页数: {}", result.getTotal(), result.getPages());
        return result;
    }
}
```

---

## 💡 实现细节

### 1. 自动填充（自动更新时间戳）

```java
// 实体类
@Data
public class User {
    private Long id;
    private String name;
    
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}

// 元对象处理器
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    
    @Override
    public void insertFill(MetaObject metaObject) {
        this.setFieldValByName("createTime", LocalDateTime.now(), metaObject);
        this.setFieldValByName("updateTime", LocalDateTime.now(), metaObject);
    }
    
    @Override
    public void updateFill(MetaObject metaObject) {
        this.setFieldValByName("updateTime", LocalDateTime.now(), metaObject);
    }
}
```

### 2. 逻辑删除（软删除）

```java
// 配置
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted  # 逻辑删除字段
      logic-delete-value: 1  # 删除时的值
      logic-not-delete-value: 0  # 未删除时的值

// 实体类
@Data
public class User {
    private Long id;
    private String name;
    
    @TableLogic
    private Integer deleted;
}

// 使用：删除自动变成 UPDATE ... SET deleted = 1
userMapper.deleteById(1L);

// 查询自动过滤
List<User> users = userMapper.selectList(null);
// 实际 SQL: SELECT * FROM user WHERE deleted = 0
```

### 3. 性能分析插件

```java
@Configuration
public class MybatisPlusConfig {
    
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        
        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        
        // 性能分析（开发环境）
        PerformanceInterceptor performanceInterceptor = new PerformanceInterceptor();
        performanceInterceptor.setFormat(true);
        performanceInterceptor.setMaxTime(100);  // SQL 执行超过 100ms 时输出日志
        
        return interceptor;
    }
}
```

---

## 🏢 行业应用

### 1. 快速 CRUD 开发

```java
@Service
public class UserService extends ServiceImpl<UserMapper, User> {
    // 继承 ServiceImpl 就获得所有 CRUD 方法
    
    public User getUserByName(String name) {
        return this.getOne(
            new QueryWrapper<User>().eq("username", name)
        );
    }
    
    public List<User> getActiveUsers() {
        return this.list(
            new QueryWrapper<User>().eq("status", 1)
        );
    }
}
```

### 2. 批量操作优化

```java
@Service
public class BulkOperationService {
    
    @Autowired
    private UserMapper userMapper;
    
    // 批量插入（性能更好）
    public void bulkInsert(List<User> users) {
        this.saveBatch(users, 100);  // 每 100 条一批
    }
    
    // 批量更新
    public void bulkUpdate(List<User> users) {
        this.updateBatchById(users, 100);
    }
}
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.1</version>
</dependency>
```

### 2. 配置

```yaml
mybatis-plus:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.xkcoding.entity
  configuration:
    map-underscore-to-camel-case: true
  global-config:
    db-config:
      id-type: ASSIGN_ID  # 自动分配 ID
```

### 3. 使用

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {
}

@Service
public class UserService extends ServiceImpl<UserMapper, User> {
}

// Controller 中使用
@RestController
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/users")
    public List<User> listUsers() {
        return userService.list();
    }
}
```

---

## 🧪 测试

```java
@SpringBootTest
class UserServiceTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    public void testInsert() {
        User user = new User();
        user.setName("Alice");
        userService.save(user);
        
        assertNotNull(user.getId());
    }
    
    @Test
    public void testQuery() {
        List<User> users = userService.list(
            new QueryWrapper<User>().eq("name", "Alice")
        );
        assertTrue(users.size() > 0);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| BaseMapper | 基础 CRUD 接口 |
| QueryWrapper | 条件查询构造器 |
| LambdaQueryWrapper | 类型安全的查询 |
| IService | Service 层接口 |
| ServiceImpl | Service 实现基类 |
| 自动填充 | MetaObjectHandler |
| 逻辑删除 | @TableLogic 注解 |
| 分页 | Page 和 IPage |

---

**恭喜！** 🎉 你已经掌握了 MyBatis Plus 的核心功能！
