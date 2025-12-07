# 统一异常处理 - 完全学习指南

## 📚 简介

在企业级应用中，异常处理不仅是捕获异常，更重要的是如何优雅地处理异常并向客户端返回有意义的错误信息。这个 demo 展示了如何通过全局异常处理器实现统一的错误响应格式。

### 为什么学习这个？
- 🎯 **理解异常处理的最佳实践**
- 🎯 **实现统一的错误响应格式**
- 🎯 **学会自定义异常和异常处理**
- 🎯 **提升应用的用户体验和可维护性**

### 核心价值
- **一致性**：所有错误都返回统一的格式
- **可读性**：错误信息清晰易懂
- **调试性**：日志记录完整的异常堆栈
- **安全性**：不向外暴露系统内部细节

---

## 🎯 核心概念详解

### 1. 异常处理的进化路径

```
阶段 1: 没有异常处理
├─ 问题: 服务器返回 500，前端不知道发生了什么
└─ 结果: 用户体验极差，调试困难

阶段 2: 在每个方法中处理异常
├─ 问题: 代码冗余，处理方式不统一
└─ 结果: 容易出现遗漏

阶段 3: 全局异常处理器（推荐）
├─ 优点: 中央处理，格式统一，代码简洁
└─ 结果: 专业的错误处理机制

阶段 4: 微服务异常处理
├─ 增加: 异常链路追踪、限流熔断等
└─ 工具: Resilience4j, Hystrix 等
```

### 2. 统一响应格式

**REST API 的响应应该包含：**

```json
{
  "code": 200,                    // 业务状态码
  "message": "操作成功",           // 用户友好的提示信息
  "data": {                        // 实际的数据
    "id": 1,
    "name": "Alice"
  },
  "timestamp": "2024-01-01T10:00:00.000Z"  // 请求时间
}
```

**异常响应：**

```json
{
  "code": 400,
  "message": "用户名不能为空",
  "error": "ValidationException",
  "details": {
    "field": "name",
    "constraint": "NotBlank"
  },
  "timestamp": "2024-01-01T10:00:00.000Z"
}
```

### 3. Spring MVC 异常处理的几种方式

#### 方式 1：@ExceptionHandler（方法级）
```java
@RestController
@RequestMapping("/users")
public class UserController {
    
    // 只在该控制器中处理该异常
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiResponse> handleUserNotFound(UserNotFoundException e) {
        ApiResponse response = ApiResponse.error(404, "用户不存在");
        return ResponseEntity.status(404).body(response);
    }
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

#### 方式 2：@ControllerAdvice（全局级）
```java
@RestControllerAdvice  // 相当于 @ControllerAdvice + @ResponseBody
@Slf4j
public class GlobalExceptionHandler {
    
    // 处理自定义异常
    @ExceptionHandler(UserNotFoundException.class)
    public ApiResponse handleUserNotFound(UserNotFoundException e) {
        log.warn("用户不存在: {}", e.getMessage());
        return ApiResponse.error(404, e.getMessage());
    }
    
    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    public ApiResponse handleBusiness(BusinessException e) {
        log.error("业务异常: {}", e.getMessage());
        return ApiResponse.error(400, e.getMessage());
    }
    
    // 处理 Spring 的验证异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.joining(", "));
        
        return ApiResponse.error(400, "参数验证失败: " + message);
    }
    
    // 处理所有未捕获的异常
    @ExceptionHandler(Exception.class)
    public ApiResponse handleUnexpected(Exception e) {
        log.error("未预期的异常", e);
        return ApiResponse.error(500, "服务器内部错误，请稍后重试");
    }
}
```

#### 方式 3：ErrorController（处理 Spring 层之外的错误）
```java
@RestController
public class CustomErrorController implements ErrorController {
    
    @RequestMapping("/error")
    public ApiResponse handleError(HttpServletRequest request) {
        Integer status = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        String message = (String) request.getAttribute(RequestDispatcher.ERROR_MESSAGE);
        
        return ApiResponse.error(status, message != null ? message : "请求错误");
    }
}
```

### 4. 自定义异常体系

```java
// 基础异常类
public class BaseException extends RuntimeException {
    private Integer code;
    
    public BaseException(Integer code, String message) {
        super(message);
        this.code = code;
    }
    
    public Integer getCode() {
        return code;
    }
}

// 业务异常
public class BusinessException extends BaseException {
    public BusinessException(String message) {
        super(400, message);
    }
}

// 资源不存在异常
public class ResourceNotFoundException extends BaseException {
    public ResourceNotFoundException(String resource, Long id) {
        super(404, resource + " 不存在: " + id);
    }
}

// 认证异常
public class UnauthorizedException extends BaseException {
    public UnauthorizedException() {
        super(401, "请先登录");
    }
}

// 授权异常
public class ForbiddenException extends BaseException {
    public ForbiddenException() {
        super(403, "没有访问权限");
    }
}

// 使用示例
@Service
public class UserService {
    public User getUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户", id));
    }
    
    public void deleteUser(Long id) {
        if (id == null || id <= 0) {
            throw new BusinessException("用户 ID 不合法");
        }
        userRepository.deleteById(id);
    }
}
```

### 5. 统一响应类

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ApiResponse<T> {
    private Integer code;           // 业务状态码
    private String message;         // 提示信息
    private T data;                // 实际数据
    private String error;          // 异常类名
    private Map<String, Object> details;  // 错误详情
    private LocalDateTime timestamp;      // 时间戳
    
    // 成功响应
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .code(200)
            .message("操作成功")
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ApiResponse<T> success(T data, String message) {
        return ApiResponse.<T>builder()
            .code(200)
            .message(message)
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    // 失败响应
    public static <T> ApiResponse<T> error(Integer code, String message) {
        return ApiResponse.<T>builder()
            .code(code)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ApiResponse<T> error(Integer code, String message, String error) {
        return ApiResponse.<T>builder()
            .code(code)
            .message(message)
            .error(error)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

---

## 💡 实现细节深入分析

### 1. 异常处理的执行顺序

```
1. 请求到达 Controller
   ↓
2. 执行 Controller 方法，如果抛出异常
   ↓
3. Spring 先寻找 @ControllerAdvice 中的 @ExceptionHandler
   ↓
4. 如果没有找到，寻找 Controller 内的 @ExceptionHandler
   ↓
5. 如果还没有找到，抛给 Spring 的异常处理器
   ↓
6. 最后由 ErrorController 处理
```

### 2. 异常类型的处理优先级

```java
@RestControllerAdvice
public class ExceptionHandler {
    
    // 优先级从高到低排序
    
    // 1. 最具体的异常（子类）
    @ExceptionHandler(UserNotFoundException.class)
    public ApiResponse handleUserNotFound(UserNotFoundException e) { }
    
    // 2. 通用业务异常
    @ExceptionHandler(BusinessException.class)
    public ApiResponse handleBusiness(BusinessException e) { }
    
    // 3. Spring 的验证异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse handleValidation(MethodArgumentNotValidException e) { }
    
    // 4. 最通用的异常（父类）
    @ExceptionHandler(Exception.class)
    public ApiResponse handleUnexpected(Exception e) { }
}
```

### 3. 异常日志记录

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ApiResponse handleUserNotFound(UserNotFoundException e) {
        // 不记录日志，这是正常的业务流程
        return ApiResponse.error(404, e.getMessage());
    }
    
    @ExceptionHandler(BusinessException.class)
    public ApiResponse handleBusiness(BusinessException e) {
        // warn 级别，可以但不需要立即处理
        log.warn("业务异常: {}", e.getMessage());
        return ApiResponse.error(400, e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    public ApiResponse handleUnexpected(Exception e) {
        // error 级别，记录完整的堆栈
        log.error("未预期的异常", e);
        return ApiResponse.error(500, "服务器内部错误");
    }
}
```

---

## 🏢 行业应用举例

### 1. 电商订单异常处理

```java
@RestControllerAdvice
@Slf4j
public class OrderExceptionHandler {
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ApiResponse<Void> handleOrderNotFound(OrderNotFoundException e) {
        return ApiResponse.error(404, "订单不存在");
    }
    
    @ExceptionHandler(InsufficientInventoryException.class)
    public ApiResponse<Void> handleInsufficientInventory(InsufficientInventoryException e) {
        return ApiResponse.error(400, "库存不足: " + e.getMessage());
    }
    
    @ExceptionHandler(PaymentFailedException.class)
    public ApiResponse<Void> handlePaymentFailed(PaymentFailedException e) {
        log.error("支付失败: {}", e.getMessage());
        return ApiResponse.error(400, "支付失败，请重试");
    }
}
```

### 2. API 限流异常

```java
@RestControllerAdvice
public class RateLimitExceptionHandler {
    
    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<ApiResponse> handleRateLimit(RateLimitExceededException e) {
        // 返回 429 状态码表示请求过于频繁
        ApiResponse response = ApiResponse.error(429, "请求过于频繁，请稍后再试");
        return ResponseEntity.status(429).body(response);
    }
}
```

---

## 🚀 快速开始指南

### 1. 创建响应格式类

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ApiResponse<T> {
    private Integer code;
    private String message;
    private T data;
    private LocalDateTime timestamp;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .code(200)
            .message("success")
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ApiResponse<T> error(Integer code, String message) {
        return ApiResponse.<T>builder()
            .code(code)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

### 2. 创建全局异常处理器

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(Exception.class)
    public ApiResponse<Void> handleException(Exception e) {
        log.error("异常发生", e);
        return ApiResponse.error(500, "服务器内部错误");
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<Void> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(Collectors.joining(", "));
        
        return ApiResponse.error(400, message);
    }
}
```

### 3. 在 Controller 中使用

```java
@RestController
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public ApiResponse<User> getUser(@PathVariable Long id) {
        User user = userService.getUserById(id);
        return ApiResponse.success(user);
    }
    
    @PostMapping
    public ApiResponse<User> createUser(@Valid @RequestBody UserRequest request) {
        User user = userService.createUser(request);
        return ApiResponse.success(user, "用户创建成功");
    }
}
```

---

## 🧪 详细测试指南

### 测试 1：正常请求

```bash
curl http://localhost:8080/users/1
```

**响应：**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "name": "Alice"
  },
  "timestamp": "2024-01-01T10:00:00"
}
```

### 测试 2：资源不存在

```bash
curl http://localhost:8080/users/999
```

**响应：**
```json
{
  "code": 404,
  "message": "用户不存在",
  "timestamp": "2024-01-01T10:00:00"
}
```

### 测试 3：参数验证错误

```bash
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"","email":"invalid"}'
```

**响应：**
```json
{
  "code": 400,
  "message": "name: 名字不能为空, email: 邮箱格式不正确",
  "timestamp": "2024-01-01T10:00:00"
}
```

---

## 🛠️ 开发实践指南

### 实践 1：自定义异常类

```java
public class ResourceNotFoundException extends RuntimeException {
    private String resource;
    private Object identifier;
    
    public ResourceNotFoundException(String resource, Object identifier) {
        super(String.format("%s 不存在: %s", resource, identifier));
        this.resource = resource;
        this.identifier = identifier;
    }
}

@RestControllerAdvice
public class ResourceNotFoundHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ApiResponse<Void> handleNotFound(ResourceNotFoundException e) {
        return ApiResponse.error(404, e.getMessage());
    }
}
```

### 实践 2：异常响应增强

```java
@Data
@Builder
public class ErrorResponse {
    private Integer code;
    private String message;
    private String error;  // 异常类名
    private Map<String, String> fieldErrors;  // 字段级别的错误
    private String stackTrace;  // 开发环境下包含堆栈
    private LocalDateTime timestamp;
}

@RestControllerAdvice
@Slf4j
public class EnhancedExceptionHandler {
    
    @Value("${app.env:production}")
    private String environment;
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ErrorResponse handleValidation(MethodArgumentNotValidException e) {
        Map<String, String> fieldErrors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        
        ErrorResponse response = ErrorResponse.builder()
            .code(400)
            .message("验证失败")
            .error(e.getClass().getSimpleName())
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
        
        // 开发环境下包含堆栈信息
        if ("dev".equals(environment)) {
            response.setStackTrace(getStackTrace(e));
        }
        
        return response;
    }
}
```

---

## 📊 常见问题解决

### Q1: 异常被 catch 后该怎么处理？
**A:** 遵循原则：
- 能处理就处理，转换为业务异常
- 不能处理就重新抛出
- 不要吞掉异常

```java
// ❌ 错误做法
try {
    userService.saveUser(user);
} catch (Exception e) {
    // 什么都不做
}

// ✅ 正确做法 1: 转换异常
try {
    userService.saveUser(user);
} catch (DataAccessException e) {
    throw new BusinessException("数据库异常");
}

// ✅ 正确做法 2: 重新抛出
try {
    userService.saveUser(user);
} catch (Exception e) {
    log.error("保存用户失败", e);
    throw e;
}
```

### Q2: 如何处理异步方法中的异常？
**A:** 使用 AsyncUncaughtExceptionHandler
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步方法异常: {} - {}", method.getName(), ex.getMessage(), ex);
        };
    }
}
```

---

## 📖 扩展学习路径

1. **消息验证** → Bean Validation（JSR-303）
2. **自定义验证器** → @Valid 和 Validator
3. **问题详情 API** → RFC 7807 标准
4. **异常链路追踪** → Sleuth + Zipkin
5. **demo-log-aop** → 结合 AOP 记录完整的请求/响应

---

**恭喜！** 🎉 你已经掌握了异常处理的最佳实践。现在你的应用能够优雅地处理各种异常情况了！
