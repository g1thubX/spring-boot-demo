# WebSocket 实时通信 - 完全学习指南

## 📚 简介

WebSocket 是一种网络通信协议，提供了双向通信的能力。与 HTTP 不同，WebSocket 建立连接后可以保持长连接，实现服务器主动推送消息给客户端，适合实时聊天、实时通知等场景。

### 为什么学习这个？
- 🎯 **理解 WebSocket 的原理和应用场景**
- 🎯 **掌握服务器推送消息的能力**
- 🎯 **实现实时聊天、通知等功能**
- 🎯 **理解连接管理和心跳机制**

---

## 🎯 核心概念

### 1. HTTP vs WebSocket

```
HTTP (Polling - 轮询)
Client: 每 3 秒请求一次服务器
Server: 有数据就返回，没数据返回空
问题: 浪费带宽，延迟高

WebSocket (长连接)
Client ←→ Server  (建立 WebSocket 连接)
Client ←┬→ Server (双向实时通信)
Server可以主动推送数据给 Client
优点: 低延迟，高效率
```

### 2. WebSocket 生命周期

```
客户端建立连接
    ↓
@OnOpen - 连接打开
    ↓
@OnMessage - 接收消息
    ↓ (可能多次)
@OnClose - 连接关闭
    ↓
@OnError - 出现异常
```

### 3. Spring WebSocket 实现

```java
// 1. 配置 WebSocket 端点
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myWebSocketHandler(), "/ws/chat")
            .setAllowedOrigins("*");
    }
    
    @Bean
    public MyWebSocketHandler myWebSocketHandler() {
        return new MyWebSocketHandler();
    }
}

// 2. 定义 WebSocket 处理器
@Component
public class MyWebSocketHandler extends TextWebSocketHandler {
    
    private List<WebSocketSession> sessions = new CopyOnWriteArrayList<>();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        log.info("WebSocket 连接已建立: {}", session.getId());
        sessions.add(session);
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        log.info("收到消息: {}", message.getPayload());
        
        // 广播消息给所有客户端
        for (WebSocketSession s : sessions) {
            if (s.isOpen()) {
                try {
                    s.sendMessage(message);
                } catch (IOException e) {
                    log.error("发送消息失败", e);
                }
            }
        }
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        log.info("WebSocket 连接已关闭: {}", session.getId());
        sessions.remove(session);
    }
}

// 3. 客户端 JavaScript
var ws = new WebSocket("ws://localhost:8080/ws/chat");

ws.onopen = function() {
    console.log("连接已建立");
    ws.send("Hello Server");
};

ws.onmessage = function(event) {
    console.log("收到消息: " + event.data);
};

ws.onerror = function(event) {
    console.log("连接出错: " + event);
};

ws.onclose = function() {
    console.log("连接已关闭");
};
```

### 4. STOMP 协议（更高级的消息协议）

```java
// STOMP 支持消息队列、订阅/发布等功能

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketMessageConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/stomp")
            .setAllowedOrigins("*")
            .withSockJS();  // SockJS 提供降级支持
    }
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // 客户端订阅的前缀
        config.enableSimpleBroker("/topic", "/queue");
        // 客户端发送的前缀
        config.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
public class ChatController {
    
    // 客户端发送: /app/chat
    @MessageMapping("/chat")
    @SendTo("/topic/messages")  // 广播给所有订阅 /topic/messages 的客户端
    public ChatMessage sendMessage(ChatMessage message) {
        message.setTimestamp(LocalDateTime.now());
        return message;
    }
    
    // 发送给特定用户
    @MessageMapping("/private")
    public void sendPrivateMessage(PrivateMessage message, Principal principal) {
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(),  // 收件人
            "/queue/messages",
            message
        );
    }
}

// 客户端 JavaScript (使用 STOMP)
var stompClient = null;

function connect() {
    var socket = new SockJS('/ws/stomp');
    stompClient = Stomp.over(socket);
    
    stompClient.connect({}, function(frame) {
        console.log('已连接: ' + frame.version);
        
        // 订阅消息
        stompClient.subscribe('/topic/messages', function(message) {
            showMessage(JSON.parse(message.body));
        });
    });
}

function sendMessage() {
    stompClient.send("/app/chat", {}, JSON.stringify({
        from: 'Alice',
        content: 'Hello'
    }));
}
```

---

## 💡 实现细节

### 1. 心跳检测（Keep-alive）

```java
@Component
public class HeartbeatHandler extends TextWebSocketHandler {
    
    private static final String PING = "PING";
    private static final String PONG = "PONG";
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        String payload = message.getPayload();
        
        if (PING.equals(payload)) {
            try {
                session.sendMessage(new TextMessage(PONG));
            } catch (IOException e) {
                log.error("发送 PONG 失败", e);
            }
        } else {
            // 处理实际消息
        }
    }
}

// 客户端定期发送 PING
setInterval(function() {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send("PING");
    }
}, 30000);  // 每 30 秒
```

### 2. 错误处理和重连

```java
public class RobustWebSocketHandler extends TextWebSocketHandler {
    
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) {
        log.error("WebSocket 异常: {}", exception.getMessage());
        
        try {
            session.close();
        } catch (IOException e) {
            log.error("关闭连接失败", e);
        }
    }
}

// 客户端重连
var reconnectAttempts = 0;
var maxReconnectAttempts = 5;
var reconnectDelay = 3000;

function connectWithRetry() {
    try {
        ws = new WebSocket("ws://localhost:8080/ws/chat");
        ws.onopen = () => {
            reconnectAttempts = 0;  // 重置计数
        };
        ws.onclose = () => {
            if (reconnectAttempts < maxReconnectAttempts) {
                reconnectAttempts++;
                setTimeout(connectWithRetry, reconnectDelay);
            }
        };
    } catch (e) {
        console.error("连接失败: " + e);
    }
}
```

---

## 🏢 行业应用

### 1. 实时聊天应用

```java
@Controller
public class ChatRoomController {
    
    @MessageMapping("/chat/{roomId}")
    @SendTo("/topic/chat/{roomId}")
    public ChatMessage chat(@DestinationVariable String roomId, ChatMessage message) {
        return message;
    }
}
```

### 2. 实时通知系统

```java
@Service
public class NotificationService {
    
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    public void notifyUser(Long userId, String message) {
        messagingTemplate.convertAndSendToUser(
            userId.toString(),
            "/queue/notifications",
            message
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
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

### 2. 启用 WebSocket

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(webSocketHandler(), "/ws")
            .setAllowedOrigins("*");
    }
    
    @Bean
    public WebSocketHandler webSocketHandler() {
        return new MyHandler();
    }
}
```

---

## 🧪 测试

使用 WebSocket 测试工具（如 Postman）或在浏览器控制台：

```javascript
var ws = new WebSocket("ws://localhost:8080/ws");
ws.send("Hello");
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| WebSocket | 双向长连接 |
| STOMP | 基于 WebSocket 的消息协议 |
| SockJS | WebSocket 降级方案 |
| 心跳检测 | 保持连接活跃 |
| 重连机制 | 自动重新连接 |

---

**恭喜！** 🎉 你已经掌握了 WebSocket 实时通信！
