# Spring Security RBAC - 完全学习指南

## 📚 简介

Spring Security 是 Spring 生态中最强大的安全框架，提供了认证（Authentication）和授权（Authorization）两大核心功能。这个 demo 展示了如何使用 Spring Security 实现基于角色的访问控制（RBAC）。

### 为什么学习这个？
- 🎯 **理解认证和授权的区别**
- 🎯 **掌握 Spring Security 的核心流程**
- 🎯 **实现 RBAC 权限模型**
- 🎯 **学会使用 JWT 保护 API**

### 核心价值
- **安全性**：保护应用和数据
- **灵活性**：支持多种认证方式
- **易用性**：声明式安全配置
- **可扩展性**：支持自定义认证和授权

---

## 🎯 核心概念详解

### 1. 认证 vs 授权

```
认证 (Authentication) - 你是谁？
├─ 用户身份验证
├─ 过程: 输入用户名密码 -> 验证 -> 登录成功
└─ 结果: SecurityContext 中存储当前用户信息

授权 (Authorization) - 你能做什么？
├─ 权限验证
├─ 过程: 检查用户是否有权执行操作
└─ 结果: 允许或拒绝访问
```

### 2. Spring Security 执行流程

```
HTTP 请求
  ↓
FilterChain (过滤链)
  ├─ SecurityContextPersistenceFilter (恢复 SecurityContext)
  ├─ UsernamePasswordAuthenticationFilter (用户密码认证)
  ├─ ExceptionTranslationFilter (异常处理)
  ├─ FilterSecurityInterceptor (权限检查)
  └─ ...其他过滤器...
  ↓
DispatcherServlet
  ↓
Handler (Controller)
  ↓
HttpResponse
```

### 3. 核心组件

| 组件 | 说明 |
|-----|------|
| SecurityContext | 存储当前用户认证信息 |
| AuthenticationManager | 认证管理器，执行认证 |
| GrantedAuthority | 权限，代表用户的权限 |
| UserDetails | 用户详情接口，用户信息载体 |
| UserDetailsService | 加载用户详情的接口 |

### 4. RBAC（基于角色的访问控制）

```
用户 (User)
  ↓
角色 (Role)
  ↓
权限 (Permission)
  ↓
资源 (Resource)

实现步骤：
1. 用户 (user_id)
2. 用户与角色关联 (user_role: user_id <-> role_id)
3. 角色与权限关联 (role_permission: role_id <-> permission_id)
4. 权限对应特定的资源操作
```

### 5. 权限配置方式

#### a) 方法级权限控制
```java
@Service
public class UserService {
    
    // 只有 ROLE_ADMIN 可以调用
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
    
    // 只有 ROLE_USER 和 ROLE_ADMIN 可以调用
    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")
    public User getUser(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }
    
    // 检查用户是否拥有 user:read 权限
    @PreAuthorize("hasAuthority('user:read')")
    public List<User> listUsers() {
        return userRepository.findAll();
    }
    
    // SpEL 表达式：允许用户访问自己的信息
    @PostAuthorize("returnObject.username == authentication.principal.username")
    public User getUserInfo(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }
}
```

#### b) URL 级权限控制
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()  // 公开接口
                .antMatchers("/admin/**").hasRole("ADMIN")  // 需要 ADMIN 角色
                .antMatchers("/user/**").hasAnyRole("USER", "ADMIN")  // 需要 USER 或 ADMIN 角色
                .anyRequest().authenticated()  // 其他请求需要认证
            .and()
                .formLogin()  // 表单登录
                .loginPage("/login")
                .loginProcessingUrl("/login")
                .usernameParameter("username")
                .passwordParameter("password")
            .and()
                .logout()  // 登出
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login");
    }
}
```

### 6. 用户加载和密码处理

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("用户不存在: " + username));
        
        // 加载用户的权限
        Collection<GrantedAuthority> authorities = user.getRoles()
            .stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toList());
        
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            true,  // 是否启用
            true,  // 是否过期
            true,  // 凭证是否过期
            true,  // 是否被锁定
            authorities
        );
    }
}

@Configuration
public class SecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        // 使用 BCrypt 加密，安全性更好
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder);
        
        return new ProviderManager(authProvider);
    }
}
```

### 7. JWT 认证

```java
// JWT 工具类
@Component
@Slf4j
public class JwtTokenProvider {
    
    @Value("${jwt.secret:your-secret-key}")
    private String jwtSecret;
    
    @Value("${jwt.expiration:86400}")  // 24 小时
    private int jwtExpirationMs;
    
    public String generateToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationMs * 1000L);
        
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public String getUsernameFromToken(String token) {
        return Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.error("JWT 验证失败: {}", e.getMessage());
            return false;
        }
    }
}

// JWT 过滤器
@Component
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain filterChain) 
        throws ServletException, IOException {
        try {
            // 从请求头获取 token
            String jwt = getJwtFromRequest(request);
            
            if (jwt != null && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromToken(jwt);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                
                // 创建认证对象
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                
                // 存储到 SecurityContext
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            log.error("JWT 认证失败: {}", e.getMessage());
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

---

## 💡 实现细节深入分析

### 1. 密码加密方案

```
❌ 不应该使用
├─ 明文存储：不安全
├─ MD5 哈希：已被破解
└─ SHA1 哈希：已被破解

✅ 应该使用
├─ BCrypt：自适应强度，推荐
├─ PBKDF2：标准算法
├─ SCrypt：高成本算法
└─ Argon2：最新推荐
```

```java
// BCrypt 使用
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        // strength: 4-31，数值越大越安全，但耗时越长
        return new BCryptPasswordEncoder(12);
    }
}

// 注册用户时加密密码
@Service
public class UserService {
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    public User registerUser(String username, String password) {
        User user = new User();
        user.setUsername(username);
        user.setPassword(passwordEncoder.encode(password));  // 加密
        return userRepository.save(user);
    }
}

// 验证时 Spring Security 自动比对
```

### 2. 权限提取和存储

```java
@Service
@Transactional
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RoleRepository roleRepository;
    
    // 为用户分配角色
    public void assignRoleToUser(Long userId, String roleName) {
        User user = userRepository.findById(userId).orElseThrow();
        Role role = roleRepository.findByName(roleName).orElseThrow();
        user.getRoles().add(role);
        userRepository.save(user);
    }
    
    // 获取用户的权限列表
    public Set<String> getUserPermissions(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        return user.getRoles()
            .stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(Permission::getCode)
            .collect(Collectors.toSet());
    }
}
```

### 3. 动态权限刷新

```java
@Component
@Slf4j
public class PermissionCache {
    
    @Autowired
    private RedisTemplate<String, Set<String>> redisTemplate;
    
    @Autowired
    private UserService userService;
    
    private static final String CACHE_PREFIX = "user:permissions:";
    
    public Set<String> getPermissions(Long userId) {
        String cacheKey = CACHE_PREFIX + userId;
        Set<String> cached = redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            return cached;
        }
        
        // 从数据库加载
        Set<String> permissions = userService.getUserPermissions(userId);
        
        // 缓存 1 小时
        redisTemplate.opsForValue().set(cacheKey, permissions, Duration.ofHours(1));
        
        return permissions;
    }
    
    // 权限变更时清除缓存
    public void invalidatePermissions(Long userId) {
        redisTemplate.delete(CACHE_PREFIX + userId);
    }
}
```

---

## 🏢 行业应用举例

### 1. 多层级权限管理

```
总管理员 (ROLE_SUPER_ADMIN)
├─ 系统管理员 (ROLE_ADMIN)
│  ├─ 用户管理权限
│  ├─ 角色管理权限
│  └─ 权限管理权限
├─ 部门管理员 (ROLE_DEPT_ADMIN)
│  └─ 部门内用户管理权限
└─ 普通用户 (ROLE_USER)
   └─ 个人信息修改权限
```

### 2. 第三方登录集成

```java
@Configuration
@EnableOAuth2Sso
public class OAuth2Config extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .anyRequest().authenticated()
            .and()
                .oauth2Login();  // 启用 OAuth2 登录
    }
}
```

---

## 🚀 快速开始指南

### 1. 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

### 2. SecurityConfig 配置

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()  // 禁用 CSRF（API 不需要）
            .authorizeRequests()
                .antMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            .and()
                .httpBasic();  // HTTP Basic 认证
    }
    
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 🧪 详细测试指南

### 测试 1：登录和获取 Token

```bash
# 登录获取 token
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}'

# 响应
{
  "token": "eyJhbGciOiJIUzUxMiJ9..."
}
```

### 测试 2：使用 Token 访问受保护资源

```bash
# 带 token 请求
curl -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9..." \
  http://localhost:8080/api/users

# 不带 token 会被拒绝
curl http://localhost:8080/api/users
# 响应: 401 Unauthorized
```

---

## 🛠️ 开发实践指南

### 实践 1：自定义权限检查

```java
@Component
public class PermissionChecker {
    
    public boolean canDeleteUser(Long userId) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) return false;
        
        // 管理员可以删除任何用户
        if (auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }
        
        // 用户只能删除自己
        String username = auth.getName();
        User currentUser = userRepository.findByUsername(username).orElse(null);
        return currentUser != null && currentUser.getId().equals(userId);
    }
}

@RestController
public class UserController {
    @Autowired
    private PermissionChecker permissionChecker;
    
    @DeleteMapping("/users/{id}")
    public void deleteUser(@PathVariable Long id) {
        if (!permissionChecker.canDeleteUser(id)) {
            throw new ForbiddenException("无权删除此用户");
        }
        userService.deleteUser(id);
    }
}
```

---

## 📊 常见问题解决

### Q1: 用户登录后，其他请求仍需输入密码？
**A:** 使用 JWT Token 或 Session，并在每个请求中携带

### Q2: 权限改变后，用户需要重新登录？
**A:** 不需要，使用权限缓存，权限改变时清除缓存

### Q3: CORS 跨域请求被拦截？
**A:** 配置 CORS
```java
@Configuration
public class CorsConfig {
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("*")
                    .allowedMethods("*");
            }
        };
    }
}
```

---

**恭喜！** 🎉 你已经掌握了 Spring Security 和 RBAC 的核心概念，可以构建安全的应用了！
