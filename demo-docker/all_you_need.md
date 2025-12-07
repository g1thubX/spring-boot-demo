# Docker 容器化 - 完全学习指南

## 📚 简介

Docker 是一个容器化平台，将应用及其依赖打包成镜像，可以在任何地方快速部署。这个 demo 展示了如何将 Spring Boot 应用 Docker 化。

### 为什么学习这个？
- 🎯 **一致性部署：开发、测试、生产环境完全一致**
- 🎯 **快速部署：秒级启动应用**
- 🎯 **资源隔离：每个容器独立的环境**
- 🎯 **简化运维：易于扩展、更新、维护**

---

## 🎯 核心概念

### 1. Docker 基础

```
镜像 (Image)
  ├─ 应用 + 依赖 + JVM
  └─ 只读，可复用

容器 (Container)
  ├─ 镜像的运行实例
  └─ 隔离的环境

Dockerfile
  └─ 构建镜像的脚本
```

### 2. Dockerfile 编写

```dockerfile
# 使用官方 Java 镜像作为基础
FROM openjdk:11-jre-slim

# 设置工作目录
WORKDIR /app

# 复制 JAR 文件到容器
COPY target/spring-boot-demo-helloworld.jar app.jar

# 暴露端口
EXPOSE 8080

# 启动应用
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3. 构建和运行

```bash
# 构建镜像
docker build -t spring-boot-demo:1.0 .

# 运行容器
docker run -d \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  --name demo \
  spring-boot-demo:1.0

# 查看日志
docker logs -f demo

# 进入容器
docker exec -it demo /bin/bash

# 停止容器
docker stop demo

# 删除容器
docker rm demo
```

### 4. Docker Compose（多容器编排）

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Spring Boot 应用
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/mydb
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=root
    depends_on:
      - mysql
      - redis

  # MySQL 数据库
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mydb
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"

  # Redis 缓存
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mysql-data:
```

### 5. 最佳实践

```dockerfile
# 多阶段构建（减小镜像大小）
# 阶段 1：构建阶段
FROM maven:3.8-openjdk-11 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# 阶段 2：运行阶段（使用小的基础镜像）
FROM openjdk:11-jre-slim
WORKDIR /app

# 只复制构建结果，不复制源代码
COPY --from=builder /app/target/*.jar app.jar

# 非 root 用户运行（安全性）
RUN useradd -m app && chown -R app:app /app
USER app

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 💡 实现细节

### 1. 优化构建过程

```yaml
# .dockerignore（排除不必要的文件）
target/
.git
.gitignore
.idea
*.iml
*.log
.DS_Store
.mvn
mvnw*
```

### 2. 健康检查

```dockerfile
FROM openjdk:11-jre-slim

WORKDIR /app
COPY target/app.jar .

EXPOSE 8080

# 健康检查（容器编排系统会定期检查）
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3. 环境变量和配置

```bash
# 运行时传入环境变量
docker run -e SPRING_PROFILES_ACTIVE=prod \
           -e SPRING_DATASOURCE_PASSWORD=secret \
           spring-boot-demo:1.0

# 或使用 .env 文件
docker run --env-file .env spring-boot-demo:1.0
```

---

## 🏢 行业应用

### 1. Kubernetes 部署

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: app
        image: spring-boot-demo:1.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### 2. CI/CD 集成

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Build Docker image
        run: docker build -t spring-boot-demo:latest .
      
      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker push spring-boot-demo:latest
```

---

## 🚀 快速开始

### 1. 创建 Dockerfile

```dockerfile
FROM openjdk:11-jre-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2. 构建镜像

```bash
mvn clean package
docker build -t my-app:1.0 .
```

### 3. 运行容器

```bash
docker run -d -p 8080:8080 my-app:1.0
```

---

## 🧪 测试

```bash
# 检查镜像
docker images

# 运行容器
docker run --rm -p 8080:8080 my-app:1.0

# 访问应用
curl http://localhost:8080/demo/hello

# 查看日志
docker logs <container-id>
```

---

## 📖 关键知识点

| 知识点 | 说明 |
|------|------|
| Image | Docker 镜像 |
| Container | Docker 容器 |
| Dockerfile | 镜像构建脚本 |
| Registry | 镜像仓库 |
| docker-compose | 多容器编排 |
| Kubernetes | 容器编排平台 |

---

**恭喜！** 🎉 你已经掌握了 Docker 容器化的基础！
