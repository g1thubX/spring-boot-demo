# MongoDB 文档数据库 - 完全学习指南

## 📚 简介

MongoDB 是一个 NoSQL 文档数据库，以 JSON 格式存储数据。相比关系型数据库，MongoDB 提供了灵活的 schema、强大的查询能力和水平扩展能力。适合存储非结构化数据、日志、缓存等。

### 为什么学习这个？
- 🎯 **灵活 Schema：无需预定义表结构**
- 🎯 **快速开发：JSON 格式，与应用天然匹配**
- 🎯 **横向扩展：分片支持 PB 级数据**
- 🎯 **强大查询：丰富的查询操作符和聚合框架**

---

## 🎯 核心概念

### 1. MongoDB vs 关系型数据库

| 概念 | 关系型数据库 | MongoDB |
|-----|-----------|---------|
| Database | Database | Database |
| Table | Table | Collection |
| Row | Row | Document |
| Column | Column | Field |
| Index | Index | Index |
| Primary Key | ID | _id |

### 2. 基本操作

```java
@Configuration
public class MongoConfig {
    
    @Bean
    public MongoTemplate mongoTemplate(MongoClient mongoClient, 
                                      MongoProperties properties) {
        return new MongoTemplate(mongoClient, properties.getDatabase());
    }
}

@Service
@Slf4j
public class UserMongoService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    // 保存文档
    public User save(User user) {
        return mongoTemplate.save(user);
    }
    
    // 根据 ID 查询
    public User findById(String id) {
        return mongoTemplate.findById(id, User.class);
    }
    
    // 查询所有
    public List<User> findAll() {
        return mongoTemplate.findAll(User.class);
    }
    
    // 条件查询
    public User findByUsername(String username) {
        Query query = new Query(Criteria.where("username").is(username));
        return mongoTemplate.findOne(query, User.class);
    }
    
    // 更新
    public void update(String id, User user) {
        Query query = new Query(Criteria.where("_id").is(id));
        Update update = new Update()
            .set("username", user.getUsername())
            .set("email", user.getEmail())
            .set("updateTime", new Date());
        
        mongoTemplate.updateFirst(query, update, User.class);
    }
    
    // 删除
    public void delete(String id) {
        Query query = new Query(Criteria.where("_id").is(id));
        mongoTemplate.remove(query, User.class);
    }
}
```

### 3. 实体和注解

```java
@Data
@Document(collection = "users")  // 对应 MongoDB 的集合
public class User {
    
    @Id
    private String id;  // MongoDB 自动生成的 _id
    
    private String username;
    
    @Field("email_address")  // 对应数据库中的 email_address 字段
    private String email;
    
    @Indexed(unique = true)  // 建立唯一索引
    private String phone;
    
    private Integer age;
    
    @CreatedDate  // 创建时间，自动赋值
    private LocalDateTime createTime;
    
    @LastModifiedDate  // 修改时间，自动更新
    private LocalDateTime updateTime;
    
    // 嵌套对象
    private Address address;
    
    // 数组
    private List<String> hobbies;
}

@Data
class Address {
    private String city;
    private String street;
    private String zipCode;
}
```

### 4. 复杂查询

```java
@Service
public class AdvancedQueryService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    // 多条件查询
    public List<User> complexQuery(String keyword, Integer minAge, Integer maxAge) {
        Query query = new Query();
        
        Criteria criteria = new Criteria();
        criteria.and("username").regex(keyword)  // 模糊搜索
                .and("age").gte(minAge).lte(maxAge);  // 范围
        
        query.addCriteria(criteria);
        
        return mongoTemplate.find(query, User.class);
    }
    
    // 分页查询
    public Page<User> pageQuery(Integer pageNo, Integer pageSize) {
        Query query = new Query();
        
        long total = mongoTemplate.count(query, User.class);
        
        query.skip((long) (pageNo - 1) * pageSize)
             .limit(pageSize)
             .with(Sort.by(Sort.Order.desc("createTime")));
        
        List<User> users = mongoTemplate.find(query, User.class);
        
        return new PageImpl<>(users, PageRequest.of(pageNo - 1, pageSize), total);
    }
    
    // 聚合查询
    public List<Map> groupByCity() {
        Aggregation aggregation = Aggregation.newAggregation(
            Aggregation.group("address.city")
                .count().as("count")
                .push("username").as("users"),
            Aggregation.sort(Sort.Direction.DESC, "count")
        );
        
        AggregationResults<Map> results = mongoTemplate
            .aggregate(aggregation, User.class, Map.class);
        
        return results.getMappedResults();
    }
}
```

### 5. Spring Data MongoDB Repository

```java
// 使用 Repository 接口快速开发
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    
    // 自定义查询方法
    User findByUsername(String username);
    
    List<User> findByAgeBetween(Integer minAge, Integer maxAge);
    
    List<User> findByAddressCityOrderByCreateTimeDesc(String city);
    
    // 使用 @Query 自定义查询
    @Query("{'username': ?0}")
    User findByUsernameCustom(String username);
}

// 使用
@Service
public class UserServiceWithRepository {
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUserByUsername(String username) {
        return userRepository.findByUsername(username);
    }
    
    public Page<User> getUsersByAgeRange(Integer minAge, Integer maxAge, 
                                        Pageable pageable) {
        List<User> users = userRepository.findByAgeBetween(minAge, maxAge);
        // 再做分页处理
        return null;
    }
}
```

---

## 💡 实现细节

### 1. 索引优化

```java
@Configuration
public class MongoIndexConfig {
    
    @Bean
    public MongoTemplate mongoTemplate(MongoClient mongoClient,
                                      MongoProperties properties) {
        MongoTemplate template = new MongoTemplate(mongoClient, 
                                                   properties.getDatabase());
        
        // 创建索引
        template.indexOps(User.class)
            .ensureIndex(new Index().on("username", Sort.Direction.ASC)
                .unique());
        
        template.indexOps(User.class)
            .ensureIndex(new Index().on("email", Sort.Direction.ASC));
        
        template.indexOps(User.class)
            .ensureIndex(new Index().on("createTime", Sort.Direction.DESC));
        
        return template;
    }
}
```

### 2. 事务支持（MongoDB 4.0+）

```java
@Service
public class TransactionalMongoService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Transactional
    public void transferBalance(String userId1, String userId2, 
                               BigDecimal amount) {
        // MongoDB 事务，要么全部成功，要么全部失败
        User user1 = mongoTemplate.findById(userId1, User.class);
        User user2 = mongoTemplate.findById(userId2, User.class);
        
        user1.setBalance(user1.getBalance().subtract(amount));
        user2.setBalance(user2.getBalance().add(amount));
        
        mongoTemplate.save(user1);
        mongoTemplate.save(user2);
    }
}
```

---

## 🏢 行业应用

### 1. 日志存储

```java
@Data
@Document(collection = "logs")
public class Log {
    @Id
    private String id;
    private String level;
    private String message;
    private LocalDateTime timestamp;
    private String service;
}

// 快速存储和查询日志
```

### 2. 用户行为追踪

```java
@Data
@Document(collection = "user_events")
public class UserEvent {
    @Id
    private String id;
    private String userId;
    private String eventType;  // 点击、浏览、购买等
    private Map<String, Object> properties;  // 灵活的事件属性
    private LocalDateTime timestamp;
}
```

---

## 🚀 快速开始

### 1. 启动 MongoDB

```bash
docker run -d --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  mongo:latest
```

### 2. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

### 3. 配置

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://admin:password@localhost:27017/mydatabase
      # 或分开配置
      host: localhost
      port: 27017
      database: mydatabase
      username: admin
      password: password
```

---

## 🧪 测试

```java
@SpringBootTest
class MongoDBTest {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Test
    public void testInsertAndFind() {
        User user = new User();
        user.setUsername("Alice");
        
        mongoTemplate.save(user);
        
        User found = mongoTemplate.findById(user.getId(), User.class);
        assertNotNull(found);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Document | MongoDB 的文档（JSON） |
| Collection | 文档集合，相当于表 |
| MongoTemplate | 操作 MongoDB 的模板类 |
| Repository | 数据访问接口 |
| Query | 查询条件构造器 |
| Aggregation | 聚合框架 |
| Index | 索引优化 |

---

**恭喜！** 🎉 你已经掌握了 MongoDB 的核心功能！
