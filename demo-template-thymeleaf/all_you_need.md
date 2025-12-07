# Thymeleaf 模板引擎 - 完全学习指南

## 📚 简介

Thymeleaf 是一个现代的 Java 模板引擎，特别为 Web 和独立环境设计。相比 JSP，Thymeleaf 更简洁、易维护；相比 Freemarker，Thymeleaf 更适合 Web 开发。

### 为什么学习这个？
- 🎯 **原生支持 Spring：深度集成 Spring MVC**
- 🎯 **HTML5 友好：模板就是有效的 HTML**
- 🎯 **IDE 友好：IDE 可以识别 Thymeleaf 表达式**
- 🎯 **前后端分离：前端可独立打开预览**

---

## 🎯 核心概念

### 1. Thymeleaf 基础语法

```html
<!-- 变量表达式 -->
<p th:text="${user.name}">用户名</p>

<!-- 选择表达式 -->
<div th:object="${user}">
    <p th:text="*{name}">名字</p>
    <p th:text="*{email}">邮箱</p>
</div>

<!-- 条件判断 -->
<div th:if="${user.age >= 18}">
    <p>成年人</p>
</div>

<!-- 循环 -->
<table>
    <tr th:each="user : ${users}">
        <td th:text="${user.name}">名字</td>
        <td th:text="${user.email}">邮箱</td>
    </tr>
</table>

<!-- 链接生成 -->
<a th:href="@{/users/{id}(id=${user.id})}">查看用户</a>

<!-- 消息国际化 -->
<p th:text="#{welcome.message(${user.name})}">欢迎</p>
```

### 2. 在 Spring Boot 中使用

```java
@Configuration
public class ThymeleafConfig {
    
    @Bean
    public ThymeleafViewResolver thymeleafViewResolver(
            SpringTemplateEngine templateEngine) {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        resolver.setTemplateEngine(templateEngine);
        resolver.setCharacterEncoding("UTF-8");
        return resolver;
    }
}

@Controller
public class UserController {
    
    @GetMapping("/users")
    public String list(Model model) {
        List<User> users = userService.listUsers();
        model.addAttribute("users", users);
        return "users/list";  // 加载 templates/users/list.html
    }
    
    @GetMapping("/users/{id}")
    public String detail(@PathVariable Long id, Model model) {
        User user = userService.getUser(id);
        model.addAttribute("user", user);
        return "users/detail";
    }
}
```

### 3. 模板文件示例

```html
<!-- templates/users/list.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>用户列表</title>
</head>
<body>
    <h1>用户列表</h1>
    
    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>名字</th>
                <th>邮箱</th>
                <th>操作</th>
            </tr>
        </thead>
        <tbody>
            <tr th:each="user : ${users}">
                <td th:text="${user.id}">1</td>
                <td th:text="${user.name}">Alice</td>
                <td th:text="${user.email}">alice@example.com</td>
                <td>
                    <a th:href="@{/users/{id}(id=${user.id})}">查看</a>
                    <a th:href="@{/users/{id}/edit(id=${user.id})}">编辑</a>
                </td>
            </tr>
        </tbody>
    </table>
</body>
</html>
```

### 4. 常用内置对象

```html
<!-- 上下文相关 -->
<p th:text="${#request.getParameter('name')}">请求参数</p>

<!-- URL 相关 -->
<a th:href="@{/users(page=1,size=10)}">第一页</a>

<!-- 日期相关 -->
<p th:text="${#dates.format(user.createTime, 'yyyy-MM-dd')}">日期</p>

<!-- 字符串相关 -->
<p th:text="${#strings.toUpperCase(user.name)}">大写</p>
<p th:text="${#strings.substring(user.email, 0, 5)}">截取</p>

<!-- 数字相关 -->
<p th:text="${#numbers.formatDecimal(price, 1, 2)}">格式化数字</p>

<!-- 列表相关 -->
<p th:text="${#lists.size(users)}">列表大小</p>

<!-- 对象相关 -->
<p th:text="${#objects.nullSafe(user.name, 'unknown')}">null 安全</p>
```

---

## 💡 实现细节

### 1. 模板继承和片段

```html
<!-- templates/layout.html (基础模板) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:block="title">默认标题</title>
</head>
<body>
    <header th:fragment="header">
        <h1>我的网站</h1>
    </header>
    
    <main th:block="content">
        <!-- 子页面的内容插入这里 -->
    </main>
    
    <footer th:fragment="footer">
        <p>&copy; 2024 版权所有</p>
    </footer>
</body>
</html>

<!-- templates/users/list.html (子页面) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      th:replace="layout :: html">
<head>
    <title th:block="title">用户列表</title>
</head>
<body>
    <main th:block="content">
        <h2>用户列表</h2>
        <!-- 用户列表内容 -->
    </main>
</body>
</html>
```

### 2. 条件和循环

```html
<!-- if/else 条件 -->
<div th:if="${user.age >= 18}">成年</div>
<div th:unless="${user.age >= 18}">未成年</div>
<div th:switch="${user.status}">
    <p th:case="1">激活</p>
    <p th:case="0">禁用</p>
    <p th:case="*">未知</p>
</div>

<!-- 循环 -->
<table>
    <tr th:each="user, stat : ${users}"
        th:class="${stat.even} ? 'even' : 'odd'">
        <td th:text="${stat.count}">序号</td>
        <td th:text="${user.name}">名字</td>
        <td th:text="${stat.first ? '第一行' : ''}">标记</td>
    </tr>
</table>
```

### 3. 集成 Spring 表达式

```html
<!-- Spring Security 集成 -->
<div sec:authorize="isAuthenticated()">
    <p>欢迎, <span sec:authentication="principal.username">用户</span></p>
</div>

<div sec:authorize="hasRole('ADMIN')">
    <a href="/admin">管理</a>
</div>

<!-- Spring 的属性访问 -->
<p th:text="${@propertyResolver.getProperty('app.name')}">应用名</p>

<!-- 方法调用 -->
<p th:text="${userService.getUserCount()}">用户总数</p>
```

---

## 🏢 行业应用

### 1. 后台管理系统

```html
<!-- 动态表单 -->
<form th:action="@{/users}" th:method="post" th:object="${user}">
    <input type="text" th:field="*{name}" placeholder="用户名">
    <input type="email" th:field="*{email}" placeholder="邮箱">
    <select th:field="*{role}">
        <option th:each="role : ${roles}" 
                th:value="${role.id}"
                th:text="${role.name}">角色</option>
    </select>
    <button type="submit">提交</button>
</form>
```

### 2. 电商产品展示

```html
<!-- 商品列表 -->
<div class="products" th:each="product : ${products}">
    <h3 th:text="${product.name}">商品名</h3>
    <p th:text="${product.description}">描述</p>
    <span th:text="|¥${#numbers.formatDecimal(product.price, 1, 2)}|">价格</span>
    <button th:onclick="|addToCart(${product.id})|">加入购物车</button>
</div>
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    encoding: UTF-8
    mode: HTML
```

### 3. 创建模板

在 `src/main/resources/templates/` 下创建 `.html` 文件

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| th:text | 设置元素文本 |
| th:if | 条件判断 |
| th:each | 循环 |
| th:href | 生成链接 |
| th:object | 选择表达式对象 |
| th:field | 表单字段绑定 |
| 内置对象 | #dates, #strings 等 |

---

**恭喜！** 🎉 你已经掌握了 Thymeleaf 模板引擎！
