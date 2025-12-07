# Neo4j 图数据库 - 完全学习指南

## 📚 简介

Neo4j 是一个图数据库，用于存储高度关联的数据。相比关系型数据库，Neo4j 在处理复杂关系时更加高效和直观。适合社交网络、推荐系统、知识图谱等应用。

### 核心特点
- 🎯 **高效关系查询：O(1) 复杂度**
- 🎯 **灵活模式：支持动态属性**
- 🎯 **Cypher 查询：直观的图查询语言**

---

## 🎯 使用方法

### 1. 定义节点和关系

```java
@Node("Person")  // 节点类型
@Data
public class Person {
    @Id
    @GeneratedValue
    private Long id;
    
    @Property  // 节点属性
    private String name;
    
    @Property
    private String email;
    
    // 一对多关系
    @Relationship(type = "FOLLOWS", direction = Relationship.Direction.OUTGOING)
    private List<Person> follows;
    
    // 关系对象（包含关系属性）
    @Relationship(type = "KNOWS")
    private List<PersonKnows> knows;
}

@RelationshipEntity(type = "KNOWS")  // 关系对象
@Data
public class PersonKnows {
    @Id
    @GeneratedValue
    private Long id;
    
    @StartNode
    private Person person1;
    
    @EndNode
    private Person person2;
    
    @Property
    private Integer strength;  // 关系强度
}
```

### 2. 操作节点

```java
@Repository
public interface PersonRepository extends Neo4jRepository<Person, Long> {
    
    // 按名字查找
    Person findByName(String name);
    
    // 自定义查询
    @Query("MATCH (p:Person) WHERE p.name = $name RETURN p")
    Person findPersonByName(@Param("name") String name);
}

@Service
public class PersonService {
    
    @Autowired
    private PersonRepository personRepository;
    
    @Autowired
    private Neo4jTemplate neo4jTemplate;
    
    // 创建节点
    public Person createPerson(String name, String email) {
        Person person = new Person();
        person.setName(name);
        person.setEmail(email);
        return personRepository.save(person);
    }
    
    // 创建关系
    public void addFollow(Long personId, Long followId) {
        Person person = personRepository.findById(personId).orElseThrow();
        Person follow = personRepository.findById(followId).orElseThrow();
        
        if (person.getFollows() == null) {
            person.setFollows(new ArrayList<>());
        }
        person.getFollows().add(follow);
        
        personRepository.save(person);
    }
}
```

### 3. 图查询

```java
@Repository
public interface PersonRepository extends Neo4jRepository<Person, Long> {
    
    // 找出所有关注者
    @Query("MATCH (p:Person)-[:FOLLOWS]->(follower:Person) " +
           "WHERE p.name = $name RETURN follower")
    List<Person> findFollowers(@Param("name") String name);
    
    // 找出共同关注者
    @Query("MATCH (p1:Person)-[:FOLLOWS]->(common:Person)<-[:FOLLOWS]-(p2:Person) " +
           "WHERE p1.name = $name1 AND p2.name = $name2 RETURN common")
    List<Person> findCommonFollowers(@Param("name1") String name1, 
                                     @Param("name2") String name2);
    
    // 推荐系统（朋友的朋友）
    @Query("MATCH (me:Person {name: $name})-[:KNOWS]-(:Person)-[:KNOWS]-(recommended:Person) " +
           "WHERE NOT (me)-[:KNOWS]-(recommended) " +
           "RETURN recommended, COUNT(*) as strength " +
           "ORDER BY strength DESC LIMIT 5")
    List<Person> recommendFriends(@Param("name") String name);
}
```

---

## 💡 实现细节

### 1. 高级查询示例

```cypher
-- 查找所有路径
MATCH path = (p1:Person)-[*]-(p2:Person)
WHERE p1.name = 'Alice' AND p2.name = 'Bob'
RETURN path

-- 最短路径
MATCH (p1:Person {name: 'Alice'}), 
      (p2:Person {name: 'Bob'})
MATCH path = shortestPath((p1)-[*]-(p2))
RETURN path

-- 统计关系强度
MATCH (p1:Person)-[k:KNOWS]->(p2:Person)
WHERE p1.name = 'Alice'
RETURN p2.name, k.strength
ORDER BY k.strength DESC
```

### 2. 性能优化

```java
@Configuration
public class Neo4jConfig {
    
    @Bean
    public Neo4jTemplate neo4jTemplate(Driver driver) {
        return new Neo4jTemplate(driver);
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| 节点 (Node) | 数据实体 |
| 关系 (Relationship) | 节点间连接 |
| 标签 (Label) | 节点分类 |
| 属性 (Property) | 节点/关系数据 |
| Cypher | 图查询语言 |

---

**恭喜！** 🎉 你已经掌握了 Neo4j 图数据库！
