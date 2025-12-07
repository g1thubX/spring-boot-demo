# Swagger API 文档 - 完全学习指南

## 📚 简介

Swagger（现为 OpenAPI）是一个强大的 API 文档框架，可以自动生成 API 文档，同时提供在线测试功能。使用 Swagger，你可以告别手动编写 API 文档的繁琐工作。

### 为什么学习这个？
- 🎯 **自动生成 API 文档，节省时间**
- 🎯 **在线测试 API，提升开发效率**
- 🎯 **前后端协作更容易**
- 🎯 **API 版本管理和维护**

---

## 🎯 核心概念

### 1. Swagger 的三个重要概念

```
OpenAPI Specification
  └─ 定义 RESTful API 的标准格式

Swagger UI
  └─ 图形化界面，展示 API 文档和测试工具

Swagger Codegen
  └─ 根据 API 定义生成客户端代码
```

### 2. 核心注解

| 注解 | 用途 |
|-----|------|
| @Api | 标记 Controller，说明其用途 |
| @ApiOperation | 说明方法的用途 |
| @ApiParam | 说明参数的含义 |
| @ApiModel | 说明实体类 |
| @ApiModelProperty | 说明实体的属性 |
| @ApiResponses | 说明多种响应情况 |

### 3. 使用示例

```java
@Api(tags = "用户管理")  // 分类标签
@RestController
@RequestMapping("/users")
public class UserController {
    
    @ApiOperation("获取用户列表")
    @GetMapping
    public List<User> listUsers() {
        return userService.listUsers();
    }
    
    @ApiOperation("根据 ID 获取用户")
    @GetMapping("/{id}")
    public User getUser(
        @ApiParam(value = "用户 ID", required = true, example = "1")
        @PathVariable Long id) {
        return userService.getUser(id);
    }
    
    @ApiOperation("创建用户")
    @PostMapping
    public User createUser(
        @ApiParam(value = "用户信息", required = true)
        @RequestBody @Valid UserRequest request) {
        return userService.createUser(request);
    }
}

@ApiModel("用户信息")
@Data
public class UserRequest {
    
    @ApiModelProperty(value = "用户名", required = true, example = "Alice")
    @NotBlank
    private String username;
    
    @ApiModelProperty(value = "邮箱", required = true, example = "alice@example.com")
    @Email
    private String email;
    
    @ApiModelProperty(value = "年龄", example = "25")
    @Min(18)
    private Integer age;
}
```

---

## 💡 实现细节

### 1. Swagger 配置

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
            .apiInfo(apiInfo())
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.xkcoding.controller"))
            .paths(PathSelectors.any())
            .build();
    }
    
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .title("API 文档")
            .description("Spring Boot Demo API 文档")
            .version("1.0.0")
            .contact(new Contact("作者", "https://example.com", "email@example.com"))
            .build();
    }
}
```

### 2. 详细的响应文档

```java
@RestController
@RequestMapping("/users")
public class UserController {
    
    @ApiOperation("获取用户详情")
    @ApiResponses({
        @ApiResponse(code = 200, message = "成功", response = User.class),
        @ApiResponse(code = 400, message = "参数错误"),
        @ApiResponse(code = 404, message = "用户不存在"),
        @ApiResponse(code = 500, message = "服务器错误")
    })
    @GetMapping("/{id}")
    public ApiResponse<User> getUser(@PathVariable Long id) {
        User user = userService.getUser(id);
        return ApiResponse.success(user);
    }
}
```

### 3. 多版本 API

```java
@Configuration
public class SwaggerConfig {
    
    @Bean
    public Docket apiV1() {
        return new Docket(DocumentationType.SWAGGER_2)
            .groupName("v1")
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.xkcoding.api.v1"))
            .build();
    }
    
    @Bean
    public Docket apiV2() {
        return new Docket(DocumentationType.SWAGGER_2)
            .groupName("v2")
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.xkcoding.api.v2"))
            .build();
    }
}
```

---

## 🏢 行业应用

### 1. 前后端分离项目

后端定义 Swagger，前端根据 Swagger 文档来调用 API。

### 2. 微服务文档聚合

使用 Swagger 聚合工具（如 Knife4j）将多个微服务的文档聚合在一起。

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>

<!-- 或者使用 Knife4j（更好看） -->
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>
```

### 2. application.yml

```yaml
springfox:
  documentation:
    swagger-ui:
      enabled: true
```

### 3. 访问文档

```
http://localhost:8080/swagger-ui.html
```

---

## 🧪 测试

在 Swagger UI 中：
1. 点击要测试的 API
2. 点击 "Try it out"
3. 输入参数
4. 点击 "Execute"
5. 查看响应

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| @Api | 标记 Controller |
| @ApiOperation | 说明方法 |
| @ApiParam | 说明参数 |
| @ApiModel | 说明实体 |
| @ApiModelProperty | 说明属性 |
| Docket | Swagger 配置类 |

---

**恭喜！** 🎉 你已经掌握了 Swagger API 文档！
