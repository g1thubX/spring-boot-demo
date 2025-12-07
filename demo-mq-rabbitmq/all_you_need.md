# RabbitMQ 消息队列 - 完全学习指南

## 📚 简介

RabbitMQ 是一个开源的消息代理，实现了 AMQP（Advanced Message Queuing Protocol）协议。在分布式系统中，RabbitMQ 用于异步处理、系统解耦和负载均衡。这个 demo 展示了如何使用 Spring Boot 与 RabbitMQ 进行各种消息传递模式。

### 为什么学习这个？
- 🎯 **理解消息队列的原理和应用场景**
- 🎯 **掌握 RabbitMQ 的核心概念：Exchange、Queue、Binding**
- 🎯 **学会四种消息传递模式**
- 🎯 **处理消息可靠性和事务问题**

### 核心价值
- **异步处理**：快速响应用户，后台异步处理
- **系统解耦**：服务之间通过消息队列通信，降低耦合度
- **削峰填谷**：应对流量突增
- **可靠交付**：确保消息不丢失

---

## 🎯 核心概念详解

### 1. RabbitMQ 核心概念

```
消息生产者 (Producer)
     ↓
  Message (消息)
     ↓
Exchange (交换机) - 路由规则
     ↓
Binding (绑定) - 连接关系
     ↓
Queue (队列) - 消息存储
     ↓
消息消费者 (Consumer)
```

**核心组件：**

| 组件 | 说明 | 类比 |
|-----|------|------|
| Connection | 到 RabbitMQ 的连接 | 网络连接 |
| Channel | 虚拟连接 | 子连接 |
| Exchange | 消息交换机 | 邮局的分类室 |
| Queue | 消息队列 | 邮箱 |
| Binding | 交换机到队列的绑定 | 分类规则 |
| RoutingKey | 路由键 | 地址标签 |

### 2. 四种消息传递模式

#### a) 直接队列（Direct）
```
生产者 --(routing_key)--> Exchange --(绑定)--> Queue --> 消费者
         "order.created"         exchange_name        queue_name
                                     ↓
                            检查 routing_key 是否匹配
```

```java
// 生产者
@Service
public class OrderProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrderCreated(Order order) {
        // 发送到交换机，指定 routing_key
        rabbitTemplate.convertAndSend(
            "order_exchange",        // 交换机名
            "order.created",         // routing_key
            order                    // 消息体
        );
    }
}

// 消费者
@Component
public class OrderConsumer {
    @RabbitListener(queues = "order_queue")
    public void handleOrderCreated(Order order) {
        log.info("处理订单: {}", order.getId());
        // 处理订单
    }
}

// 配置
@Configuration
public class RabbitConfig {
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("order_exchange", true, false);
    }
    
    @Bean
    public Queue orderQueue() {
        return new Queue("order_queue", true);
    }
    
    @Bean
    public Binding orderBinding(Queue orderQueue, DirectExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
            .to(orderExchange)
            .with("order.created");
    }
}
```

**特点：** 一对一的精确匹配

#### b) 广播（Fanout）
```
生产者 --> Exchange --(复制)--> Queue1 --> Consumer1
                      \
                       --> Queue2 --> Consumer2
                       \
                        --> Queue3 --> Consumer3
```

```java
@Service
public class EventProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishEvent(String event) {
        // Fanout 不需要 routing_key
        rabbitTemplate.convertAndSend("event_fanout", "", event);
    }
}

@Configuration
public class FanoutConfig {
    @Bean
    public FanoutExchange eventExchange() {
        return new FanoutExchange("event_fanout", true, false);
    }
    
    @Bean
    public Queue emailQueue() {
        return new Queue("email_queue");
    }
    
    @Bean
    public Queue smsQueue() {
        return new Queue("sms_queue");
    }
    
    @Bean
    public Queue logQueue() {
        return new Queue("log_queue");
    }
    
    @Bean
    public Binding emailBinding(Queue emailQueue, FanoutExchange eventExchange) {
        return BindingBuilder.bind(emailQueue).to(eventExchange);
    }
    
    @Bean
    public Binding smsBinding(Queue smsQueue, FanoutExchange eventExchange) {
        return BindingBuilder.bind(smsQueue).to(eventExchange);
    }
    
    @Bean
    public Binding logBinding(Queue logQueue, FanoutExchange eventExchange) {
        return BindingBuilder.bind(logQueue).to(eventExchange);
    }
}

@Component
public class MultiConsumer {
    @RabbitListener(queues = "email_queue")
    public void sendEmail(String message) {
        log.info("发送邮件: {}", message);
    }
    
    @RabbitListener(queues = "sms_queue")
    public void sendSms(String message) {
        log.info("发送短信: {}", message);
    }
    
    @RabbitListener(queues = "log_queue")
    public void writeLog(String message) {
        log.info("记录日志: {}", message);
    }
}
```

**特点：** 一对多的广播

#### c) 主题（Topic）
```
生产者 --(routing_key)--> Exchange --(通配符匹配)--> Queues --> 消费者
      "order.created.gold"    topic         
      "order.created.silver"
      "order.shipped.any"
         |
         ├─ order.created.* 匹配 gold 和 silver
         ├─ order.*.* 匹配所有
         └─ order.shipped.# 匹配所有带 shipped 的
```

```java
@Service
public class TopicProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrderEvent(String level, String event) {
        String routingKey = "order." + level + "." + event;
        rabbitTemplate.convertAndSend("order_topic", routingKey, 
            "Order event: " + routingKey);
    }
}

@Configuration
public class TopicConfig {
    @Bean
    public TopicExchange orderTopic() {
        return new TopicExchange("order_topic", true, false);
    }
    
    @Bean
    public Queue vipOrderQueue() {
        return new Queue("vip_order_queue");
    }
    
    @Bean
    public Queue normalOrderQueue() {
        return new Queue("normal_order_queue");
    }
    
    @Bean
    public Binding vipBinding(Queue vipOrderQueue, TopicExchange orderTopic) {
        return BindingBuilder.bind(vipOrderQueue)
            .to(orderTopic)
            .with("order.vip.*");  // 匹配所有 VIP 订单
    }
    
    @Bean
    public Binding normalBinding(Queue normalOrderQueue, TopicExchange orderTopic) {
        return BindingBuilder.bind(normalOrderQueue)
            .to(orderTopic)
            .with("order.normal.*");  // 匹配所有普通订单
    }
}

@Component
public class TopicConsumer {
    @RabbitListener(queues = "vip_order_queue")
    public void handleVipOrder(String message) {
        log.info("处理 VIP 订单: {}", message);
    }
    
    @RabbitListener(queues = "normal_order_queue")
    public void handleNormalOrder(String message) {
        log.info("处理普通订单: {}", message);
    }
}
```

**特点：** 灵活的通配符匹配

#### d) 延迟队列（Dead Letter）
```
生产者 --> Queue (with TTL)
           ↓ (消息过期，未被消费)
      Dead Letter Exchange
           ↓
      Delay Queue
           ↓
      消费者（延迟后才消费）
```

```java
@Configuration
public class DelayQueueConfig {
    // 原始队列
    @Bean
    public Queue taskQueue() {
        return QueueBuilder.durable("task_queue")
            .withArgument("x-message-ttl", 5000)  // 5秒过期
            .withArgument("x-dead-letter-exchange", "task_dlx")
            .build();
    }
    
    // Dead Letter Exchange
    @Bean
    public DirectExchange taskDlx() {
        return new DirectExchange("task_dlx", true, false);
    }
    
    // 延迟队列
    @Bean
    public Queue delayQueue() {
        return new Queue("task_delay_queue");
    }
    
    @Bean
    public Binding delayBinding(Queue delayQueue, DirectExchange taskDlx) {
        return BindingBuilder.bind(delayQueue)
            .to(taskDlx)
            .with("task");
    }
}

@Component
public class DelayConsumer {
    @RabbitListener(queues = "task_delay_queue")
    public void handleDelayedTask(String message) {
        log.info("处理延迟任务: {}", message);
        // 消息在消费前已经被延迟 5 秒了
    }
}
```

---

## 💡 实现细节深入分析

### 1. 消息可靠性保证

```
三个环节可能丢失消息：
1. 生产者 -> RabbitMQ
   ├─ 解决: 发送确认 (Publisher Confirm)
   └─ 配置: spring.rabbitmq.publisher-confirms=true

2. RabbitMQ 存储
   ├─ 解决: 持久化 (声明队列时 durable=true)
   └─ 配置: @Bean Queue persistent queue

3. RabbitMQ -> 消费者
   ├─ 解决: 消费确认 (Consumer Ack)
   └─ 配置: spring.rabbitmq.listener.simple.acknowledge-mode=MANUAL
```

```java
@Configuration
public class RabbitConfig {
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        
        // 发送确认
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("消息发送失败: {}", cause);
                // 重试或记录
            }
        });
        
        // 未送达交换机时的回调
        template.setReturnsCallback(returned -> {
            log.error("消息未被交换机接收: {}", returned.getReplyText());
        });
        
        return template;
    }
}

@Configuration
public class RabbitListenerConfig {
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = 
            new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);  // 手动确认
        factory.setConcurrentConsumers(1);
        factory.setMaxConcurrentConsumers(5);
        return factory;
    }
}

@Component
@Slf4j
public class ReliableConsumer {
    @RabbitListener(queues = "reliable_queue")
    public void handleMessage(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        try {
            log.info("收到消息: {}", message);
            // 处理消息
            
            // 手动确认（告诉 RabbitMQ 已处理）
            channel.basicAck(tag, false);
        } catch (Exception e) {
            try {
                // 处理失败，重新入队
                channel.basicNack(tag, false, true);
            } catch (IOException ex) {
                log.error("Nack 失败", ex);
            }
        }
    }
}
```

### 2. 消息转换

```java
@Configuration
public class RabbitConfig {
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}

@Service
public class UserProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendUser(User user) {
        // 自动序列化为 JSON
        rabbitTemplate.convertAndSend("user_exchange", "user.created", user);
    }
}

@Component
public class UserConsumer {
    @RabbitListener(queues = "user_queue")
    public void handleUser(User user) {
        // 自动反序列化为对象
        log.info("收到用户: {} ({})", user.getName(), user.getEmail());
    }
}
```

---

## 🏢 行业应用举例

### 1. 订单处理流程

```
用户下单 --> 立即返回订单号 (HTTP 200)
         ↓
      消息队列
         ↓
    扣库存 --> 生成发货单 --> 调用支付 --> 通知用户
```

### 2. 日志收集

```
应用 --> RabbitMQ --> Logstash --> Elasticsearch --> Kibana
                 \
                  --> 本地磁盘备份
```

---

## 🚀 快速开始指南

### 1. 启动 RabbitMQ

```bash
# Docker 方式
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

# 访问管理界面
http://localhost:15672  # 用户名/密码: guest/guest
```

### 2. 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 3. application.yml 配置

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    publisher-confirms: true
    publisher-returns: true
    listener:
      simple:
        acknowledge-mode: MANUAL
        concurrency: 1
        max-concurrency: 5
```

---

## 🧪 详细测试指南

### 测试 1：Direct 模式

```bash
# 启动应用，发送消息
curl -X POST http://localhost:8080/messages/direct \
  -d "Hello Direct Queue"

# RabbitMQ 管理界面查看
# http://localhost:15672 -> Queues -> direct_queue -> get message
```

### 测试 2：Fanout 模式

```bash
curl -X POST http://localhost:8080/messages/fanout \
  -d "Hello Fanout"

# 所有监听 Fanout Exchange 的消费者都会收到
```

---

## 🛠️ 开发实践指南

### 实践 1：异步发送邮件

```java
@Service
public class UserService {
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public User registerUser(UserRequest request) {
        User user = new User(request);
        userRepository.save(user);
        
        // 异步发送欢迎邮件
        rabbitTemplate.convertAndSend(
            "user_event_topic",
            "user.registered",
            user
        );
        
        return user;
    }
}

@Component
public class EmailConsumer {
    @Autowired
    private EmailService emailService;
    
    @RabbitListener(queues = "email_queue")
    public void sendWelcomeEmail(User user) {
        emailService.sendWelcome(user.getEmail(), user.getName());
    }
}
```

---

## 📊 常见问题解决

### Q1: 消息重复消费怎么办？
**A:** 实现幂等性
```java
@Component
public class IdempotentConsumer {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @RabbitListener(queues = "order_queue")
    public void handleOrder(Order order) {
        String key = "processed:order:" + order.getId();
        if (redisTemplate.hasKey(key)) {
            log.info("订单已处理: {}", order.getId());
            return;
        }
        
        // 处理订单
        processOrder(order);
        
        // 标记为已处理
        redisTemplate.opsForValue().set(key, "true", Duration.ofHours(24));
    }
}
```

### Q2: 如何处理消费失败？
**A:** 实现重试机制

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          initial-interval: 1000  # 1秒
          multiplier: 1.0
          max-attempts: 3  # 最多重试 3 次
```

---

**恭喜！** 🎉 你已经掌握了 RabbitMQ 的核心概念和四种消息传递模式！
