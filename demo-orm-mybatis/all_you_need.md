# MyBatis ORM - 完全学习指南

## 📚 简介

MyBatis 是一个灵活的 ORM 框架，与 JPA 不同，它更接近 SQL，让你能够精确控制 SQL 查询。Spring Boot 集成 MyBatis 使得开发更加便捷。

### 为什么学习这个？
- 🎯 **与 JPA 对比，掌握两种 ORM 方案**
- 🎯 **学会手写 SQL，获得更大的控制权**
- 🎯 **掌握动态 SQL，应对复杂查询**
- 🎯 **了解 MyBatis 的性能优化**

---

## 🎯 核心概念

### 1. MyBatis 执行流程

```
Mapper 接口
  ↓
XML Mapper 文件（或注解）
  ↓
SqlSessionFactory
  ↓
SqlSession
  ↓
JDBC
  ↓
数据库
```

### 2. Mapper 定义方式

#### a) 注解方式（简单查询）

```java
@Mapper
public interface UserMapper {
    
    @Insert("INSERT INTO orm_user(name, email) VALUES(#{name}, #{email})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    void insert(User user);
    
    @Select("SELECT * FROM orm_user WHERE id = #{id}")
    User selectById(@Param("id") Long id);
    
    @Update("UPDATE orm_user SET name=#{name} WHERE id=#{id}")
    int update(User user);
    
    @Delete("DELETE FROM orm_user WHERE id=#{id}")
    int delete(@Param("id") Long id);
}
```

#### b) XML 方式（复杂查询）

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.xkcoding.orm.mybatis.mapper.UserMapper">
    
    <!-- 结果集映射 -->
    <resultMap id="userMap" type="com.xkcoding.orm.mybatis.entity.User">
        <id column="id" property="id"/>
        <result column="name" property="name"/>
        <result column="email" property="email"/>
        <result column="create_time" property="createTime"/>
    </resultMap>
    
    <!-- 查询所有 -->
    <select id="selectAll" resultMap="userMap">
        SELECT * FROM orm_user
    </select>
    
    <!-- 条件查询 -->
    <select id="selectByCondition" parameterType="map" resultMap="userMap">
        SELECT * FROM orm_user WHERE 1=1
        <if test="name != null">
            AND name LIKE #{name}
        </if>
        <if test="email != null">
            AND email = #{email}
        </if>
    </select>
    
    <!-- 动态 INSERT -->
    <insert id="insert" parameterType="User" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO orm_user
        <trim prefix="(" suffix=")" suffixOverrides=",">
            <if test="name != null">name,</if>
            <if test="email != null">email,</if>
        </trim>
        VALUES
        <trim prefix="(" suffix=")" suffixOverrides=",">
            <if test="name != null">#{name},</if>
            <if test="email != null">#{email},</if>
        </trim>
    </insert>
</mapper>
```

### 3. 动态 SQL 标签

| 标签 | 说明 |
|-----|------|
| `<if>` | 条件判断 |
| `<choose/when/otherwise>` | 多条件选择 |
| `<foreach>` | 循环 |
| `<where>` | 智能 WHERE 子句 |
| `<set>` | 智能 SET 子句 |
| `<trim>` | 字符串处理 |

### 4. 参数传递

```java
@Mapper
public interface UserMapper {
    
    // 单个参数
    User selectById(Long id);
    
    // 多个参数需要 @Param
    User selectByNameAndEmail(@Param("name") String name, @Param("email") String email);
    
    // 对象参数（属性对应 XML 中的 #{name}, #{email}）
    void insert(User user);
    
    // Map 参数
    List<User> selectByCondition(Map<String, Object> condition);
    
    // List 参数
    void insertBatch(@Param("users") List<User> users);
}
```

---

## 💡 实现细节

### 1. 事务管理

```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    @Transactional
    public void transferData(Long sourceId, Long targetId) {
        User source = userMapper.selectById(sourceId);
        User target = userMapper.selectById(targetId);
        
        source.setStatus(0);  // 禁用
        target.setStatus(1);  // 启用
        
        userMapper.update(source);
        userMapper.update(target);
        
        // 自动提交
        // 如果出现异常，自动回滚
    }
}
```

### 2. 结果集映射

```xml
<!-- 复杂对象映射 -->
<resultMap id="orderMap" type="Order">
    <id column="order_id" property="id"/>
    <result column="order_date" property="orderDate"/>
    
    <!-- 一对一关联 -->
    <association property="user" javaType="User">
        <id column="user_id" property="id"/>
        <result column="user_name" property="name"/>
    </association>
    
    <!-- 一对多关联 -->
    <collection property="items" ofType="OrderItem">
        <id column="item_id" property="id"/>
        <result column="item_name" property="name"/>
    </collection>
</resultMap>

<select id="selectOrderWithDetails" resultMap="orderMap">
    SELECT 
        o.id as order_id,
        o.order_date,
        u.id as user_id,
        u.name as user_name,
        oi.id as item_id,
        oi.name as item_name
    FROM orders o
    LEFT JOIN users u ON o.user_id = u.id
    LEFT JOIN order_items oi ON o.id = oi.order_id
    WHERE o.id = #{orderId}
</select>
```

### 3. 批量操作

```java
@Mapper
public interface UserMapper {
    
    // 批量插入
    @Insert("<script>" +
            "INSERT INTO orm_user(name, email) VALUES" +
            "<foreach collection='users' item='user' separator=','>" +
            "(#{user.name}, #{user.email})" +
            "</foreach>" +
            "</script>")
    void insertBatch(@Param("users") List<User> users);
    
    // 批量删除
    void deleteBatch(@Param("ids") List<Long> ids);
}

<!-- 在 XML 中定义 -->
<delete id="deleteBatch" parameterType="list">
    DELETE FROM orm_user WHERE id IN
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</delete>
```

---

## 🏢 行业应用

### 1. 复杂业务查询

```xml
<select id="selectUserStatistics" resultType="map">
    SELECT 
        u.id,
        u.name,
        COUNT(o.id) as order_count,
        SUM(o.amount) as total_amount
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.status = 1
    GROUP BY u.id, u.name
    HAVING COUNT(o.id) > 0
    ORDER BY total_amount DESC
</select>
```

### 2. 条件动态查询

```java
@Mapper
public interface ProductMapper {
    List<Product> selectByCondition(ProductQuery query);
}

// XML
<select id="selectByCondition" parameterType="ProductQuery" resultType="Product">
    SELECT * FROM products WHERE 1=1
    <if test="categoryId != null">
        AND category_id = #{categoryId}
    </if>
    <if test="minPrice != null">
        AND price >= #{minPrice}
    </if>
    <if test="maxPrice != null">
        AND price <= #{maxPrice}
    </if>
    <if test="keyword != null">
        AND (name LIKE CONCAT('%', #{keyword}, '%') 
             OR description LIKE CONCAT('%', #{keyword}, '%'))
    </if>
    ORDER BY 
    <choose>
        <when test="sortBy == 'price'">price</when>
        <when test="sortBy == 'sales'">sales_count</when>
        <otherwise>create_time</otherwise>
    </choose>
    DESC
    LIMIT #{offset}, #{limit}
</select>
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

### 2. application.yml

```yaml
mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.xkcoding.orm.mybatis.entity
  configuration:
    map-underscore-to-camel-case: true  # 自动转换驼峰
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 打印 SQL
```

### 3. 创建 Mapper

```java
@Mapper
public interface UserMapper {
    User selectById(Long id);
    void insert(User user);
}
```

---

## 🧪 测试

```java
@SpringBootTest
class UserMapperTest {
    
    @Autowired
    private UserMapper userMapper;
    
    @Test
    public void testSelect() {
        User user = userMapper.selectById(1L);
        assertNotNull(user);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| @Mapper | 标记为 MyBatis Mapper |
| 动态 SQL | 根据条件动态生成 SQL |
| ResultMap | 字段与属性的映射 |
| TypeHandler | 自定义类型转换 |
| Plugin | 拦截器扩展 |

---

## MyBatis vs JPA 对比

| 特性 | MyBatis | JPA |
|-----|---------|-----|
| 学习曲线 | 陡 | 平 |
| SQL 控制 | 精确 | 自动 |
| 性能 | 好 | 一般 |
| 代码量 | 多 | 少 |
| 灵活性 | 高 | 低 |
| 适合场景 | 复杂查询 | CRUD 操作 |

**建议：** 简单项目用 JPA，复杂项目用 MyBatis

---

**恭喜！** 🎉 你已经掌握了 MyBatis 的核心功能！
