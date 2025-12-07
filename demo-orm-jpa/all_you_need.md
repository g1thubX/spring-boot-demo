# Spring Data JPA - 完全学习指南

## 📚 简介

JPA（Java Persistence API）是 Java 的对象关系映射（ORM）标准。Spring Data JPA 在 JPA 基础上进一步简化了数据库操作，让你能用面向对象的方式操作数据库，而无需手写复杂的 SQL。

### 为什么学习这个？
- 🎯 **理解 ORM 思想和关系映射**
- 🎯 **掌握 JPA 实体、关联、级联等核心概念**
- 🎯 **学会使用 Spring Data Repository 进行数据操作**
- 🎯 **处理一对多、多对多等复杂关系**

### 核心价值
- **面向对象**：用 Java 对象操作数据库，而非 SQL
- **少写代码**：自动生成基本 CRUD 操作
- **类型安全**：编译时检查，避免 SQL 注入
- **跨数据库**：同一套代码支持多种数据库

---

## 🎯 核心概念详解

### 1. ORM 是什么？

ORM（Object-Relational Mapping）是一种编程技术，将数据库表映射到 Java 对象：

```
关系数据库              ->         Java 对象
┌─────────────────┐              ┌──────────┐
│ 表 (Table)      │     ←→       │ 类 (Class)
│─────────────────│              │──────────│
│ 列 (Column)     │     ←→       │ 属性     │
│ 行 (Row)        │     ←→       │ 实例     │
│ 数据类型        │     ←→       │ Java类型 │
└─────────────────┘              └──────────┘

示例：
数据库表: orm_user
  ├─ id (INT, PRIMARY KEY)    ←→  Long id
  ├─ name (VARCHAR)           ←→  String name
  ├─ email (VARCHAR)          ←→  String email
  └─ status (INT)             ←→  Integer status

数据库行: [1, 'Alice', 'alice@example.com', 1]
Java对象: User(id=1, name='Alice', email='alice@example.com', status=1)
```

### 2. JPA 三个核心组件

**a) 实体类（Entity）**
```java
@Entity                    // 标记为 JPA 实体
@Table(name = "orm_user")  // 映射到哪个表
@Data                      // Lombok 生成 Getter/Setter
public class User extends AbstractAuditModel {
    @Id                    // 主键
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name")  // 列名映射
    private String name;
    
    private String email;  // 没有 @Column 会使用属性名
    
    private Integer status; // 对应数据库的 INT 类型
}
```

**b) 实体继承与审计**
```java
// 基类：存储通用的审计字段
@MappedSuperclass  // 不是数据库表，只是用来继承
@EntityListeners(AuditingEntityListener.class)
@Data
public abstract class AbstractAuditModel implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @CreatedDate  // 创建时自动赋值
    @Column(name = "create_time", nullable = false, updatable = false)
    private Date createTime;
    
    @LastModifiedDate  // 修改时自动更新
    @Column(name = "last_update_time", nullable = false)
    private Date lastUpdateTime;
}

// 具体实体类继承审计基类
@Entity
public class User extends AbstractAuditModel {
    private String name;
    private String email;
    // 自动继承 id, createTime, lastUpdateTime
}
```

**c) Repository 数据访问层**
```java
// 继承 JpaRepository 获得基本 CRUD 能力
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // User: 实体类型
    // Long: 主键类型
    
    // 基本 CRUD 由 JpaRepository 提供：
    // save(User) - 保存或更新
    // findById(Long) - 根据 ID 查询
    // findAll() - 查询所有
    // delete(User) - 删除
    // count() - 统计数量
    
    // 可以定义自定义查询方法
    User findByName(String name);
    List<User> findByEmailContaining(String email);
}
```

### 3. 实体关系映射

#### a) 一对多（One-to-Many）和多对一（Many-to-One）

```java
// 部门实体
@Entity
public class Department {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 一个部门有多个员工
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();
}

// 员工实体
@Entity
public class Employee {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 多个员工属于一个部门
    @ManyToOne
    @JoinColumn(name = "dept_id")  // 外键列名
    private Department department;
}

// 使用：
Department dept = departmentRepository.findById(1L);
List<Employee> employees = dept.getEmployees();  // 自动加载部门下的所有员工
```

#### b) 多对多（Many-to-Many）

```java
// 用户实体
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 一个用户属于多个部门
    @ManyToMany(cascade = CascadeType.ALL)
    @JoinTable(
        name = "user_department",              // 关联表名
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "dept_id")
    )
    private Collection<Department> departments = new ArrayList<>();
}

// 部门实体
@Entity
public class Department {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 被维护端，不需要 @JoinTable
    @ManyToMany(mappedBy = "departments")
    private Collection<User> users = new ArrayList<>();
}

// 使用：
User user = userRepository.findById(1L);
List<Department> depts = new ArrayList<>(user.getDepartments());
```

#### c) 一对一（One-to-One）

```java
// 员工实体
@Entity
public class Employee {
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 一个员工有一个工号卡
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "card_id")
    private EmployeeCard card;
}

// 工号卡实体
@Entity
public class EmployeeCard {
    @Id
    @GeneratedValue
    private Long id;
    
    private String cardNumber;
    
    // 被维护端
    @OneToOne(mappedBy = "card")
    private Employee employee;
}
```

### 4. 级联操作（Cascade）

级联定义了当主对象执行操作时，关联对象如何响应：

```java
@Entity
public class Blog {
    @Id
    @GeneratedValue
    private Long id;
    
    private String title;
    
    // cascade = CascadeType.ALL 表示：
    // - 保存博客时自动保存所有评论
    // - 删除博客时自动删除所有评论
    @OneToMany(mappedBy = "blog", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
}

@Entity
public class Comment {
    @Id
    @GeneratedValue
    private Long id;
    
    private String content;
    
    @ManyToOne
    @JoinColumn(name = "blog_id")
    private Blog blog;
}

// 使用：
Blog blog = new Blog();
blog.setTitle("My Blog");
blog.getComments().add(new Comment("Great post!"));
blog.getComments().add(new Comment("Very helpful!"));

blogRepository.save(blog);  // 自动保存博客和所有评论

blogRepository.delete(blog);  // 自动删除博客和所有评论
```

### 5. 关键注解总结

| 注解 | 用途 | 示例 |
|-----|------|------|
| @Entity | 标记为 JPA 实体 | @Entity |
| @Table | 指定映射的表名 | @Table(name="user") |
| @Id | 标记主键 | @Id |
| @GeneratedValue | 生成主键策略 | @GeneratedValue(strategy=GenerationType.IDENTITY) |
| @Column | 指定列名和属性 | @Column(name="email", nullable=false) |
| @OneToMany | 一对多关系 | @OneToMany(mappedBy="parent") |
| @ManyToOne | 多对一关系 | @ManyToOne |
| @ManyToMany | 多对多关系 | @ManyToMany |
| @OneToOne | 一对一关系 | @OneToOne |
| @JoinColumn | 指定外键列 | @JoinColumn(name="dept_id") |
| @JoinTable | 指定关联表 | @JoinTable(name="user_dept") |
| @CreatedDate | 创建时间自动赋值 | @CreatedDate |
| @LastModifiedDate | 修改时间自动更新 | @LastModifiedDate |

---

## 💡 实现细节深入分析

### 1. Repository 方法命名约定

Spring Data JPA 会根据方法名自动生成查询：

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // 查询单个
    User findByName(String name);
    User findByEmail(String email);
    
    // 查询多个
    List<User> findByStatus(Integer status);
    List<User> findByNameContaining(String name);  // LIKE 查询
    
    // 多条件查询
    User findByNameAndEmail(String name, String email);
    User findByNameOrEmail(String name, String email);
    
    // 复杂查询
    List<User> findByStatusNotNull();
    List<User> findByStatusIn(List<Integer> statuses);
    List<User> findByEmailStartingWith(String prefix);
    List<User> findByEmailEndingWith(String suffix);
    
    // 排序和分页
    List<User> findByStatus(Integer status, Sort sort);
    Page<User> findByStatus(Integer status, Pageable pageable);
    
    // 统计
    Long countByStatus(Integer status);
    Boolean existsByEmail(String email);
    
    // 删除
    void deleteByEmail(String email);
}
```

**主要关键词：**
- `And` / `Or` - 逻辑运算符
- `Is` / `Equals` - 相等
- `Between` - 范围
- `LessThan` / `GreaterThan` - 大小比较
- `Like` / `Containing` - 模糊查询
- `In` - 集合查询
- `NotNull` - 非空判断
- `OrderBy` - 排序

### 2. 事务管理

```java
@Service
@Transactional  // 类级别，所有方法都在事务中
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    // 自动开启事务，方法执行完自动提交
    public void transferPoints(Long fromUserId, Long toUserId, Integer points) {
        User from = userRepository.findById(fromUserId).orElseThrow();
        User to = userRepository.findById(toUserId).orElseThrow();
        
        from.setPoints(from.getPoints() - points);
        to.setPoints(to.getPoints() + points);
        
        userRepository.save(from);
        userRepository.save(to);
        // 自动提交
        // 如果中间出现异常，自动回滚
    }
    
    @Transactional(readOnly = true)  // 只读事务，优化性能
    public User getUserWithOrders(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOperation(String operation) {
        // 创建新事务，独立提交
    }
}
```

### 3. 查询方法的三种方式

**a) 方法名查询**
```java
User user = userRepository.findByName("Alice");
```

**b) @Query 注解**
```java
@Query("SELECT u FROM User u WHERE u.name = ?1 AND u.status = ?2")
List<User> findActiveUsersByName(String name, Integer status);

// 使用命名参数（更清晰）
@Query("SELECT u FROM User u WHERE u.name = :name AND u.status = :status")
List<User> findActiveUsersByNameNamed(@Param("name") String name, @Param("status") Integer status);
```

**c) 原生 SQL**
```java
@Query(value = "SELECT * FROM orm_user WHERE status = ?1", nativeQuery = true)
List<User> findActiveUsersNative(Integer status);
```

### 4. 分页查询

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public void pageQuery() {
        // 第 0 页（从 0 开始），每页 10 条，按 ID 倒序排列
        PageRequest pageRequest = PageRequest.of(
            0,                                  // 页码
            10,                                 // 每页数量
            Sort.by(Sort.Direction.DESC, "id") // 排序方式
        );
        
        Page<User> page = userRepository.findAll(pageRequest);
        
        System.out.println("总条数: " + page.getTotalElements());
        System.out.println("总页数: " + page.getTotalPages());
        System.out.println("当前页码: " + page.getNumber());
        System.out.println("当前页数据: " + page.getContent());
        System.out.println("是否有下一页: " + page.hasNext());
    }
}
```

---

## 🏢 行业应用举例

### 1. 电商产品管理

```java
@Entity
@Table(name = "product")
@Data
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private BigDecimal price;
    private Integer stock;
    
    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
    
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    private List<ProductImage> images = new ArrayList<>();
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "category")
@Data
public class Category {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "category")
    private List<Product> products = new ArrayList<>();
}

// Repository
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByCategoryAndStockGreaterThan(Category category, Integer stock);
    Page<Product> findByNameContaining(String name, Pageable pageable);
}
```

### 2. 用户权限系统

```java
@Entity
@Data
public class User {
    @Id
    @GeneratedValue
    private Long id;
    private String username;
    private String password;
    
    @ManyToMany(cascade = CascadeType.ALL)
    @JoinTable(
        name = "user_role",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}

@Entity
@Data
public class Role {
    @Id
    @GeneratedValue
    private Long id;
    private String roleName;
    
    @ManyToMany
    @JoinTable(
        name = "role_permission",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions = new HashSet<>();
}

@Entity
@Data
public class Permission {
    @Id
    @GeneratedValue
    private Long id;
    private String permissionName;
}
```

### 3. 博客评论系统

```java
@Entity
@Data
public class BlogPost {
    @Id
    @GeneratedValue
    private Long id;
    private String title;
    private String content;
    
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
    
    @CreatedDate
    private LocalDateTime createdAt;
}

@Entity
@Data
public class Comment {
    @Id
    @GeneratedValue
    private Long id;
    private String content;
    
    @ManyToOne
    @JoinColumn(name = "post_id")
    private BlogPost post;
    
    @ManyToOne
    @JoinColumn(name = "author_id")
    private User author;
    
    @CreatedDate
    private LocalDateTime createdAt;
}
```

---

## 🚀 快速开始指南

### 1. 基本设置

**pom.xml：**
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

**application.yml：**
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp
    username: root
    password: root
  jpa:
    hibernate:
      ddl-auto: create-drop  # create-drop | create | update | validate
    show-sql: true  # 打印 SQL
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL57InnoDBDialect
        format_sql: true
```

### 2. 定义实体

```java
@Entity
@Table(name = "orm_user")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(unique = true)
    private String email;
    
    private Integer age;
}
```

### 3. 创建 Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    User findByName(String name);
    List<User> findByAgeGreaterThan(Integer age);
}
```

### 4. 在 Service 中使用

```java
@Service
@Transactional
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    // 保存
    public User createUser(String name, String email) {
        User user = new User(null, name, email, 20);
        return userRepository.save(user);
    }
    
    // 查询
    public User getUserByName(String name) {
        return userRepository.findByName(name);
    }
    
    // 更新
    public User updateUser(Long id, String name) {
        User user = userRepository.findById(id).orElseThrow();
        user.setName(name);
        return userRepository.save(user);
    }
    
    // 删除
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

---

## 🧪 详细测试指南

### 测试 1：基本 CRUD

```java
@SpringBootTest
@Transactional
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testSave() {
        User user = new User(null, "Alice", "alice@example.com", 25);
        User saved = userRepository.save(user);
        
        assertNotNull(saved.getId());
        assertEquals("Alice", saved.getName());
    }
    
    @Test
    void testFindById() {
        User user = userRepository.findById(1L).orElse(null);
        assertNotNull(user);
    }
    
    @Test
    void testDelete() {
        long count = userRepository.count();
        userRepository.deleteById(1L);
        assertEquals(count - 1, userRepository.count());
    }
}
```

### 测试 2：自定义查询

```java
@Test
void testFindByName() {
    User user = userRepository.findByName("Alice");
    assertNotNull(user);
    assertEquals("alice@example.com", user.getEmail());
}

@Test
void testFindByAgeGreaterThan() {
    List<User> users = userRepository.findByAgeGreaterThan(25);
    assertTrue(users.size() > 0);
}
```

### 测试 3：分页查询

```java
@Test
void testPageQuery() {
    PageRequest pageRequest = PageRequest.of(0, 10, Sort.by("id").descending());
    Page<User> page = userRepository.findAll(pageRequest);
    
    assertEquals(10, page.getSize());
    assertTrue(page.getTotalElements() > 0);
}
```

### 测试 4：事务测试

```java
@Test
@Transactional
void testTransactional() {
    User user = new User(null, "Bob", "bob@example.com", 30);
    userRepository.save(user);
    
    // 在同一事务中，可以看到刚保存的用户
    User found = userRepository.findByName("Bob");
    assertNotNull(found);
}

@Test
void testTransactionRollback() {
    try {
        userService.createUserWithException("Charlie");
    } catch (Exception e) {
        // 异常被捕获
    }
    
    // 由于事务回滚，用户不应该被保存
    User found = userRepository.findByName("Charlie");
    assertNull(found);
}
```

---

## 🛠️ 开发实践指南

### 实践 1：实现用户权限查询

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // 通过关联表查询
    @Query("SELECT u FROM User u JOIN u.roles r WHERE r.roleName = :roleName")
    List<User> findByRoleName(@Param("roleName") String roleName);
    
    @Query("SELECT u FROM User u WHERE SIZE(u.roles) > :minRoles")
    List<User> findByMinRoles(@Param("minRoles") int minRoles);
}
```

### 实践 2：批量操作优化

```java
@Service
public class UserBatchService {
    @Autowired
    private UserRepository userRepository;
    
    @Transactional
    public void bulkCreateUsers(List<UserDTO> dtos) {
        List<User> users = dtos.stream()
            .map(dto -> new User(null, dto.getName(), dto.getEmail(), dto.getAge()))
            .collect(Collectors.toList());
        
        // 批量保存，性能更好
        userRepository.saveAll(users);
    }
}
```

### 实践 3：懒加载处理

```java
@Entity
@Data
public class Order {
    @Id
    @GeneratedValue
    private Long id;
    
    // LAZY: 只有在访问时才加载关联的用户
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    // EAGER: 立即加载订单项
    @OneToMany(mappedBy = "order", fetch = FetchType.EAGER)
    private List<OrderItem> items;
}

@Service
public class OrderService {
    public void handleLazyLoad() {
        Order order = orderRepository.findById(1L).orElse(null);
        
        // 此时 user 还未加载（如果是 LAZY）
        // 下面这行会触发加载，或者报 LazyInitializationException
        // String userName = order.getUser().getName();
        
        // 解决方案 1: 立即访问
        // 解决方案 2: 改用 EAGER
        // 解决方案 3: 使用 @Query 显式 JOIN FETCH
    }
}
```

### 实践 4：自定义审计字段

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> Optional.of(SecurityContextHolder.getContext()
            .getAuthentication()
            .getName());
    }
}

@Entity
@Data
public class BlogPost {
    @Id
    @GeneratedValue
    private Long id;
    
    @CreatedDate
    private LocalDateTime createdDate;
    
    @CreatedBy  // 自动记录创建者
    private String createdBy;
    
    @LastModifiedDate
    private LocalDateTime modifiedDate;
    
    @LastModifiedBy  // 自动记录修改者
    private String modifiedBy;
}
```

---

## 📊 常见问题解决

### Q1: LazyInitializationException
**问题：** 在 Hibernate 会话关闭后访问懒加载属性
**解决：**
```java
// 方案 1: 改为 EAGER 加载
@ManyToOne(fetch = FetchType.EAGER)

// 方案 2: 使用 JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.id = :id")
Optional<Order> findByIdWithUser(@Param("id") Long id);

// 方案 3: 在同一事务中访问
```

### Q2: N+1 查询问题
**问题：** 查询 100 个 Order，每个需要加载 User，共 101 条 SQL
**解决：**
```java
// 错误做法
List<Order> orders = orderRepository.findAll();
orders.forEach(o -> System.out.println(o.getUser().getName()));  // N+1

// 正确做法 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser();

// 正确做法 2: 手动 JOIN
@Query("SELECT o FROM Order o JOIN o.user WHERE o.id IN :ids")
List<Order> findByIdsWithUser(@Param("ids") List<Long> ids);
```

### Q3: 级联删除导致意外删除
**问题：** 删除博客时意外删除了所有评论
**解决：** 明确级联策略
```java
// 明确指定级联操作
@OneToMany(mappedBy = "blog", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<Comment> comments;  // 删除博客时不删除评论

// 或使用 orphanRemoval
@OneToMany(mappedBy = "blog", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Comment> comments;  // 只删除孤立的评论
```

---

## 📖 扩展学习路径

1. **Spring Data REST** → 自动生成 RESTful API
2. **Querydsl** → 类型安全的动态查询
3. **Spring Data JPA Specification** → 动态条件查询
4. **Hibernate 高级特性** → 缓存、连接池等
5. **多数据源** → demo-multi-datasource-jpa

---

## 🎓 关键知识点总结

| 知识点 | 关键要点 |
|------|--------|
| @Entity | 标记为 JPA 实体 |
| @Table | 指定表名 |
| @Id/@GeneratedValue | 主键和生成策略 |
| @ManyToOne | 多对一关系，包含外键 |
| @OneToMany | 一对多关系，被维护端 |
| @ManyToMany | 多对多关系，需要关联表 |
| @JoinColumn | 指定外键列名 |
| @JoinTable | 指定关联表信息 |
| cascade | 级联操作策略 |
| fetch | 加载策略（LAZY/EAGER） |
| @Transactional | 事务管理 |
| Repository 方法名 | 自动生成查询语句 |
| @Query | 自定义 JPQL 或 SQL 查询 |
| Page/Pageable | 分页查询 |

---

**恭喜！** 🎉 你已经掌握了 Spring Data JPA 的核心功能。现在你可以用优雅的面向对象方式进行数据操作了！
