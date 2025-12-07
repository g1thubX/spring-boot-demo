# 邮件发送 - 完全学习指南

## 📚 简介

在企业应用中，邮件是重要的通知和沟通方式。Spring Boot 通过 JavaMailSender 简化了邮件发送的复杂性，支持发送简单文本邮件、HTML 邮件、附件和模板邮件。

### 为什么学习这个？
- 🎯 **掌握邮件发送的完整流程**
- 🎯 **支持多种邮件类型：文本、HTML、附件**
- 🎯 **集成模板引擎发送动态邮件**
- 🎯 **处理邮件异常和重试**

---

## 🎯 核心概念

### 1. 邮件发送的三层结构

```
应用层 (Application)
  ↓
Spring Mail API (JavaMailSender)
  ↓
SMTP 协议
  ↓
邮件服务器 (如 Gmail, 企业 Exchange)
  ↓
收件人邮箱
```

### 2. 邮件配置

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: your-email@gmail.com
    password: your-app-password  # 不是你的账户密码
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
      mail.smtp.starttls.required: true
```

### 3. 邮件发送的四种方式

```java
@Service
@Slf4j
public class EmailService {
    
    @Autowired
    private JavaMailSender mailSender;
    
    // 1. 发送简单文本邮件
    public void sendSimpleEmail(String to, String subject, String content) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom("sender@example.com");
        message.setTo(to);
        message.setSubject(subject);
        message.setText(content);
        
        mailSender.send(message);
        log.info("文本邮件已发送给: {}", to);
    }
    
    // 2. 发送 HTML 邮件
    public void sendHtmlEmail(String to, String subject, String htmlContent) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        
        helper.setFrom("sender@example.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(htmlContent, true);  // true 表示 HTML
        
        mailSender.send(message);
        log.info("HTML 邮件已发送给: {}", to);
    }
    
    // 3. 发送带附件的邮件
    public void sendEmailWithAttachment(String to, String subject, String content, File attachment) 
            throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        
        helper.setFrom("sender@example.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(content);
        
        // 添加附件
        FileSystemResource file = new FileSystemResource(attachment);
        helper.addAttachment(file.getFilename(), file);
        
        mailSender.send(message);
        log.info("附件邮件已发送给: {}", to);
    }
    
    // 4. 发送模板邮件
    public void sendTemplateEmail(String to, String subject, Map<String, Object> model) 
            throws MessagingException {
        // 使用 Thymeleaf 或 Freemarker 渲染模板
        String htmlContent = renderTemplate("email-template.html", model);
        sendHtmlEmail(to, subject, htmlContent);
    }
}
```

### 4. 邮件模板

```html
<!-- resources/templates/email-template.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: Arial, sans-serif; }
        .container { width: 600px; margin: 0 auto; padding: 20px; }
        .header { background-color: #007bff; color: white; padding: 10px; }
        .content { padding: 20px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>欢迎, <span th:text="${username}"></span></h1>
        </div>
        <div class="content">
            <p>您的账户激活代码: <strong th:text="${activationCode}"></strong></p>
            <p>请在 24 小时内完成激活。</p>
        </div>
    </div>
</body>
</html>
```

---

## 💡 实现细节

### 1. 异步发送邮件

```java
@Service
public class AsyncEmailService {
    
    @Autowired
    private JavaMailSender mailSender;
    
    @Async  // 异步方法
    public void sendEmailAsync(String to, String subject, String content) {
        try {
            // 邮件发送可能耗时，使用异步避免阻塞
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom("sender@example.com");
            message.setTo(to);
            message.setSubject(subject);
            message.setText(content);
            
            mailSender.send(message);
        } catch (MailException e) {
            log.error("邮件发送失败", e);
            // 重试逻辑
        }
    }
}

// 启用异步
@SpringBootApplication
@EnableAsync
public class Application { }
```

### 2. 邮件重试机制

```java
@Service
@Slf4j
public class EmailServiceWithRetry {
    
    @Autowired
    private JavaMailSender mailSender;
    
    public void sendEmailWithRetry(String to, String subject, String content) {
        int maxRetries = 3;
        int attempt = 0;
        
        while (attempt < maxRetries) {
            try {
                SimpleMailMessage message = new SimpleMailMessage();
                message.setFrom("sender@example.com");
                message.setTo(to);
                message.setSubject(subject);
                message.setText(content);
                
                mailSender.send(message);
                log.info("邮件发送成功");
                return;
            } catch (MailException e) {
                attempt++;
                if (attempt < maxRetries) {
                    log.warn("第 {} 次邮件发送失败，准备重试...", attempt);
                    try {
                        Thread.sleep(2000 * attempt);  // 指数退避
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                    }
                } else {
                    log.error("邮件发送失败，已达到最大重试次数", e);
                }
            }
        }
    }
}
```

### 3. 邮件队列（结合 RabbitMQ）

```java
@Service
public class EmailQueueService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendEmailViaQueue(EmailRequest request) {
        // 将邮件请求放入队列
        rabbitTemplate.convertAndSend("email_queue", request);
    }
}

@Component
public class EmailConsumer {
    
    @Autowired
    private JavaMailSender mailSender;
    
    @RabbitListener(queues = "email_queue")
    public void consumeEmail(EmailRequest request) {
        // 从队列中消费邮件请求，发送邮件
        // 如果失败，自动重试
    }
}
```

---

## 🏢 行业应用

### 1. 用户注册确认邮件

```java
@Service
public class UserRegistrationService {
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private TemplateEngine templateEngine;
    
    public void registerUser(UserRequest request) {
        // 保存用户
        User user = saveUser(request);
        
        // 生成激活码
        String activationCode = generateActivationCode();
        
        // 发送激活邮件
        Context context = new Context();
        context.setVariable("username", user.getUsername());
        context.setVariable("activationCode", activationCode);
        String htmlContent = templateEngine.process("email-activation", context);
        
        emailService.sendHtmlEmail(
            user.getEmail(),
            "账户激活",
            htmlContent
        );
    }
}
```

### 2. 订单确认邮件

```java
@Service
public class OrderService {
    
    @Autowired
    private EmailService emailService;
    
    public void createOrder(OrderRequest request) {
        Order order = saveOrder(request);
        
        // 发送订单确认邮件
        String subject = "订单确认 #" + order.getId();
        String content = "您已下单，订单号为: " + order.getId();
        
        emailService.sendHtmlEmail(
            order.getCustomer().getEmail(),
            subject,
            buildOrderHtml(order)
        );
    }
}
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: your-email@gmail.com
    password: app-specific-password
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

### 3. 发送邮件

```java
@Service
public class EmailService {
    
    @Autowired
    private JavaMailSender mailSender;
    
    public void send(String to, String subject, String content) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(to);
        message.setSubject(subject);
        message.setText(content);
        
        mailSender.send(message);
    }
}
```

---

## 🧪 测试

```java
@SpringBootTest
class EmailServiceTest {
    
    @Autowired
    private EmailService emailService;
    
    @Test
    public void testSendEmail() {
        emailService.send(
            "test@example.com",
            "Test Subject",
            "Test Content"
        );
    }
}
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| JavaMailSender | Spring 邮件发送接口 |
| SimpleMailMessage | 简单邮件消息 |
| MimeMessage | MIME 格式邮件（支持 HTML、附件） |
| MimeMessageHelper | 邮件消息构建工具 |
| SMTP | 邮件发送协议 |
| 异步发送 | 使用 @Async 避免阻塞 |

---

**恭喜！** 🎉 你已经掌握了邮件发送的完整知识！
