# Kafka 消息队列 - 完全学习指南

## 📚 简介

Kafka 是一个分布式流处理平台，与 RabbitMQ 相比，Kafka 更关注高吞吐量、可扩展性和数据流处理。它广泛用于日志收集、实时数据处理、事件流等场景。

### 为什么学习这个？
- 🎯 **超高吞吐量：每秒百万级消息**
- 🎯 **可扩展性：轻松扩展到成千上万的分区**
- 🎯 **消息持久化：数据安全可靠**
- 🎯 **流处理：支持实时数据处理和分析**

---

## 🎯 核心概念

### 1. Kafka vs RabbitMQ

| 特性 | Kafka | RabbitMQ |
|-----|-------|---------|
| 吞吐量 | 百万级/秒 | 十万级/秒 |
| 延迟 | 毫秒级 | 微秒级 |
| 持久化 | 磁盘存储 | 内存为主 |
| 消费者 | 消费者组 | 队列绑定 |
| 应用 | 日志、流处理 | 任务队列、RPC |

### 2. Kafka 三个核心概念

```
Producer (生产者)
    ↓
Broker (Kafka 服务器)
├─ Topic (主题)
│  ├─ Partition 0 ─┐
│  ├─ Partition 1 ─┼─→ Consumer Group
│  └─ Partition 2 ─┘
└─ Replica (副本，高可用)

Offset (消息位移)
├─ Consumer 记录已消费的位置
└─ 支持重新消费
```

### 3. 发送和消费消息

```java
// 生产者
@Service
public class KafkaProducer {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void sendMessage(String topic, String message) {
        kafkaTemplate.send(topic, message);
    }
    
    // 发送并获取结果
    public void sendMessageWithCallback(String topic, String message) {
        ListenableFuture<SendResult<String, String>> future = 
            kafkaTemplate.send(topic, message);
        
        future.addCallback(
            result -> log.info("发送成功: {}", result.getRecordMetadata()),
            ex -> log.error("发送失败", ex)
        );
    }
}

// 消费者
@Service
@Slf4j
public class KafkaConsumer {
    
    @KafkaListener(topics = "my-topic", groupId = "my-group")
    public void consume(String message) {
        log.info("收到消息: {}", message);
    }
    
    // 消费 JSON 消息
    @KafkaListener(topics = "user-topic", groupId = "user-group")
    public void consumeUser(User user) {
        log.info("收到用户: {}", user.getName());
    }
}
```

### 4. 消费者组和分区

```
Topic: user-events (3 个分区)
├─ Partition 0 ──┐
├─ Partition 1 ──┼─→ Consumer Group: user-service
├─ Partition 2 ──┘

每个分区只能被一个消费者消费
→ 如果有 5 个消费者，只有 3 个活跃

增加分区数可以提高并发性能
增加消费者数可以加快处理速度
```

---

## 💡 实现细节

### 1. 消息顺序性保证

```java
// Kafka 保证：同一分区内的消息顺序
// 方案：所有订单消息发送到同一分区
@Service
public class OrderProducer {
    
    @Autowired
    private KafkaTemplate<String, Order> kafkaTemplate;
    
    public void sendOrder(Order order) {
        // 使用 userId 作为 key，确保同一用户的消息到同一分区
        kafkaTemplate.send("order-topic", String.valueOf(order.getUserId()), order);
    }
}

@Service
public class OrderConsumer {
    
    // 单线程消费，保证顺序
    @KafkaListener(topics = "order-topic", groupId = "order-group")
    public void consumeOrder(Order order) {
        processOrder(order);
    }
}
```

### 2. 错误处理和重试

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all  # 等待所有副本确认
      retries: 3  # 重试次数
      properties:
        linger.ms: 100  # 批处理延迟
    consumer:
      auto-offset-reset: earliest  # 从最早的消息开始消费
      enable-auto-commit: false  # 手动提交 offset
      max-poll-records: 100  # 单次最多拉取数量
```

### 3. 手动提交 Offset

```java
@Service
@Slf4j
public class ManualCommitConsumer {
    
    @KafkaListener(topics = "my-topic", groupId = "my-group")
    public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            String message = record.value();
            log.info("处理消息: {}", message);
            
            // 业务处理
            processMessage(message);
            
            // 手动提交（确保消息已处理）
            ack.acknowledge();
        } catch (Exception e) {
            log.error("处理失败，消息会重新消费", e);
            // 不提交，消息会被重新消费
        }
    }
}
```

### 4. 死信队列处理

```java
@Service
@Slf4j
public class DeadLetterConsumer {
    
    @KafkaListener(topics = "my-topic", groupId = "my-group")
    public void consume(String message) {
        try {
            processMessage(message);
        } catch (Exception e) {
            log.error("处理失败，发送到死信队列", e);
            // 发送到死信队列
            kafkaTemplate.send("dead-letter-topic", message);
        }
    }
}
```

---

## 🏢 行业应用

### 1. 日志收集

```
应用 A → Kafka → ELK 分析
应用 B ↗          ↓ 报警
应用 C ↗
```

### 2. 实时数据处理

```
用户事件 → Kafka → Spark/Flink → 实时仪表板
                              → 数据仓库
```

---

## 🚀 快速开始

### 1. 启动 Kafka

```bash
# Docker 启动
docker run -d --name kafka \
  -p 9092:9092 \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  confluentinc/cp-kafka:latest
```

### 2. 依赖

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### 3. 配置

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      group-id: my-group
```

---

## 🧪 测试

```bash
# 创建主题
kafka-topics --create --topic my-topic --bootstrap-servers localhost:9092

# 发送消息
kafka-console-producer --topic my-topic --bootstrap-servers localhost:9092

# 消费消息
kafka-console-consumer --topic my-topic --bootstrap-servers localhost:9092 --from-beginning
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Producer | 消息生产者 |
| Consumer | 消息消费者 |
| Topic | 消息主题 |
| Partition | 分区，支持并行 |
| Broker | Kafka 服务器 |
| Offset | 消息位移 |
| 消费者组 | 多个消费者组合 |

---

**恭喜！** 🎉 你已经掌握了 Kafka 的核心知识！
