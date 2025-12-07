# Freemarker 模板引擎 - 完全学习指南

## 📚 简介

Freemarker 是一个强大的 Java 模板引擎，用于生成动态内容（HTML、邮件、报告等）。它支持变量插值、条件语句、循环、自定义函数等功能，让内容展示与逻辑分离。

### 为什么学习这个？
- 🎯 **分离关注点：模板与代码分离**
- 🎯 **动态内容：支持变量、条件、循环等**
- 🎯 **多用途：可用于 Web、邮件、报告等**
- 🎯 **易于维护：非开发人员也能修改模板**

---

## 🎯 核心概念

### 1. 模板基础语法

```freemarker
<!-- 变量插值 -->
${variableName}
${user.name}
${list[0]}

<!-- 条件判断 -->
<#if user.age >= 18>
    成年人
<#else>
    未成年人
</#if>

<!-- 循环 -->
<#list users as user>
    <p>${user.name}</p>
</#list>

<!-- 内置函数 -->
${user.name?upper_case}  <!-- 大写 -->
${price?string("0.00")}  <!-- 格式化数字 -->
${date?string("yyyy-MM-dd")}  <!-- 格式化日期 -->
```

### 2. 在 Spring Boot 中使用

```java
@Configuration
public class FreemarkerConfig {
    
    @Bean
    public FreeMarkerConfigurationFactoryBean freeMarkerConfigurationFactoryBean() {
        FreeMarkerConfigurationFactoryBean bean = new FreeMarkerConfigurationFactoryBean();
        bean.setTemplateLoaderPath("classpath:/templates/");
        return bean;
    }
}

@Service
public class MailService {
    
    @Autowired
    private FreeMarkerConfigurationFactory configFactory;
    
    public String renderEmailTemplate(String templateName, Map<String, Object> model) 
            throws IOException, TemplateException {
        Configuration config = configFactory.getConfiguration();
        Template template = config.getTemplate(templateName);
        return FreeMarkerTemplateUtils.processTemplateIntoString(template, model);
    }
}

@RestController
public class PageController {
    
    @Autowired
    private MailService mailService;
    
    @GetMapping("/send-email")
    public String sendEmail() throws IOException, TemplateException {
        Map<String, Object> model = new HashMap<>();
        model.put("username", "Alice");
        model.put("activationCode", "123456");
        
        String html = mailService.renderEmailTemplate("email-template.ftl", model);
        // 发送邮件...
        
        return "邮件已发送";
    }
}
```

### 3. 模板文件示例

```freemarker
<!-- templates/email-template.ftl -->
<!DOCTYPE html>
<html>
<head>
    <title>激活账户</title>
</head>
<body>
    <h1>欢迎, ${username}!</h1>
    
    <p>您的激活码：<strong>${activationCode}</strong></p>
    
    <p>请在 24 小时内完成激活。</p>
    
    <#if products??>
        <h2>热门商品：</h2>
        <ul>
        <#list products as product>
            <li>${product.name} - ￥${product.price}</li>
        </#list>
        </ul>
    </#if>
</body>
</html>
```

### 4. 常用内置函数

```freemarker
<!-- 字符串函数 -->
${name?upper_case}  <!-- 大写 -->
${name?lower_case}  <!-- 小写 -->
${name?capitalize}  <!-- 首字母大写 -->
${name?string}      <!-- 转字符串 -->

<!-- 数字函数 -->
${price?string("0.00")}  <!-- 格式化 -->
${price?ceil}            <!-- 向上取整 -->
${price?floor}           <!-- 向下取整 -->

<!-- 日期函数 -->
${date?string("yyyy-MM-dd HH:mm:ss")}
${date?string("yyyy-MM-dd")}

<!-- 判断函数 -->
${name?? && name != ""}  <!-- 非空判断 -->
${list?has_content}      <!-- 列表非空 -->
${string?length}         <!-- 字符串长度 -->
```

---

## 💡 实现细节

### 1. 集成 Web 视图

```java
@Configuration
public class ViewConfig extends WebMvcConfigurer {
    
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // 配置 Freemarker 视图解析器
        FreeMarkerViewResolver resolver = new FreeMarkerViewResolver();
        resolver.setPrefix("/templates/");
        resolver.setSuffix(".ftl");
        resolver.setContentType("text/html;charset=UTF-8");
        registry.viewResolver(resolver);
    }
}

@Controller
public class WebController {
    
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("title", "首页");
        model.addAttribute("username", "Alice");
        return "home";  // 自动加载 templates/home.ftl
    }
}
```

### 2. 宏定义（可复用模板片段）

```freemarker
<!-- 定义宏 -->
<#macro userCard user>
    <div class="card">
        <h3>${user.name}</h3>
        <p>邮箱：${user.email}</p>
        <p>年龄：${user.age}</p>
    </div>
</#macro>

<!-- 使用宏 -->
<@userCard user=currentUser />

<!-- 带参数的宏 -->
<#macro pagination currentPage totalPages>
    <div class="pagination">
        <#list 1..totalPages as page>
            <#if page == currentPage>
                <span>${page}</span>
            <#else>
                <a href="?page=${page}">${page}</a>
            </#if>
        </#list>
    </div>
</#macro>

<@pagination currentPage=1 totalPages=5 />
```

---

## 🏢 行业应用

### 1. 邮件模板

```freemarker
<!-- templates/order-confirmation.ftl -->
<h1>订单确认</h1>
<p>订单号：${order.id}</p>
<table>
    <tr><th>商品</th><th>数量</th><th>价格</th></tr>
    <#list order.items as item>
    <tr>
        <td>${item.name}</td>
        <td>${item.quantity}</td>
        <td>¥${item.price}</td>
    </tr>
    </#list>
</table>
<p>总计：¥${order.totalAmount}</p>
```

### 2. HTML 页面

```freemarker
<!-- templates/product-list.ftl -->
<#include "header.ftl" />

<div class="products">
    <#list products as product>
        <div class="product">
            <h3>${product.name}</h3>
            <p>${product.description}</p>
            <span>¥${product.price}</span>
        </div>
    </#list>
</div>

<#include "footer.ftl" />
```

---

## 🚀 快速开始

### 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  freemarker:
    template-loader-path: classpath:/templates/
    suffix: .ftl
    charset: UTF-8
```

### 3. 创建模板

在 `src/main/resources/templates/` 下创建 `.ftl` 文件

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| 变量插值 | ${variable} |
| 条件语句 | <#if> <#else> |
| 循环 | <#list> |
| 宏 | <#macro> |
| 内置函数 | ?upper_case, ?string 等 |
| 包含 | <#include> |
| 导入 | <#import> |

---

**恭喜！** 🎉 你已经掌握了 Freemarker 模板引擎！
