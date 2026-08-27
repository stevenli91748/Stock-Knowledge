# Day 1：AWS + Java 股票应用开发环境

今天只做一件事：

> **把 AWS 云端开发环境搭好，并让第一个 Spring Boot API 在本地成功运行。**

### 今天完成后的架构

```text
你的 Mac / Windows
       │
       ├── Java 21
       ├── IntelliJ IDEA
       ├── Git
       ├── Docker
       └── AWS CLI
             │
             ▼
          GitHub
             │
             ▼
            AWS
```

---

## 1. 注册 / 准备 AWS

进入：

[AWS 官方网站](https://aws.amazon.com/?utm_source=chatgpt.com)

建议今天先**不要购买任何 AWS 服务**。

重点学习：

* AWS Account
* IAM
* Region
* Availability Zone
* Billing

### Region

你的股票应用建议选择：

**US West (Oregon) — `us-west-2`**

后面我们会统一使用：

```bash
aws configure
```

然后：

```text
Region:
us-west-2
```

---

# 2. 安装 Java 21

推荐：

**Amazon Corretto 21**

这是 AWS 提供的 OpenJDK 发行版。

[Amazon Corretto 官方页面](https://aws.amazon.com/corretto/?utm_source=chatgpt.com)

安装后检查：

```bash
java -version
```

应该看到类似：

```text
openjdk version "21.x.x"
```

再检查：

```bash
javac -version
```

应该：

```text
javac 21.x.x
```

---

# 3. 安装 IntelliJ IDEA

推荐：

**IntelliJ IDEA**

[JetBrains IntelliJ IDEA](https://www.jetbrains.com/idea/?utm_source=chatgpt.com)

创建第一个项目时：

```text
Language: Java
Build System: Maven
JDK: 21
```

---

# 4. 安装 Git

[Git 官方网站](https://git-scm.com/?utm_source=chatgpt.com)

检查：

```bash
git --version
```

例如：

```text
git version 2.x.x
```

然后设置：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
```

---

# 5. 创建 GitHub Repository

进入：

[GitHub](https://github.com/?utm_source=chatgpt.com)

创建：

```text
stock-ai-platform
```

建议：

```text
Public
```

因为以后这是你的 AI Engineer 求职项目。

---

# 6. 创建 Spring Boot 项目

打开：

[Spring Initializr](https://start.spring.io/?utm_source=chatgpt.com)

设置：

```text
Project: Maven

Language: Java

Spring Boot: 3.x

Group:
com.stockai

Artifact:
stock-backend

Name:
stock-backend

Packaging:
Jar

Java:
21
```

Dependencies：

```text
Spring Web
Spring Boot Actuator
Validation
Lombok
```

暂时**不要加入数据库和 Security**。

今天先把最基础环境跑起来。

---

# 7. 第一个 API

创建：

```text
src/main/java/com/stockai/controller/HealthController.java
```

代码：

```java
package com.stockai.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {

    @GetMapping("/api/health")
    public String health() {
        return "Stock AI Platform is running!";
    }
}
```

启动：

```bash
./mvnw spring-boot:run
```

Windows：

```bash
mvnw.cmd spring-boot:run
```

浏览器打开：

```text
http://localhost:8080/api/health
```

看到：

```text
Stock AI Platform is running!
```

说明 **Java + Spring Boot 成功**。

---

# 8. 今天必须理解的 AWS 三个概念

### IAM

管理：

```text
Who can access AWS?
What can they do?
```

以后我们的：

```text
Developer
GitHub Actions
ECS
Lambda
```

都会使用 IAM 权限。

**不要使用 AWS Root Account 做日常开发。**

---

### EC2

传统：

```text
EC2
 ↓
自己管理服务器
 ↓
安装 Java
 ↓
部署 Spring Boot
```

先知道即可。

我们的第一版不会优先使用 EC2。

---

### ECS Fargate

以后：

```text
Spring Boot
     ↓
Docker
     ↓
ECR
     ↓
ECS Fargate
```

AWS 负责服务器基础设施。

这会是我们第一版生产部署的主要方式。

---

# 9. 安装 AWS CLI

[AWS CLI 官方安装文档](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html?utm_source=chatgpt.com)

检查：

```bash
aws --version
```

然后：

```bash
aws configure
```

输入：

```text
AWS Access Key ID:
AWS Secret Access Key:
Default region name:
us-west-2
Default output format:
json
```

**不要把 Access Key / Secret Key 放进 GitHub。**

更不要写进：

```text
application.properties
```

以后我们会使用：

**AWS Secrets Manager / IAM Role**

处理密钥。

---

# 10. 今天最后做 Git

进入项目：

```bash
cd stock-backend
```

初始化：

```bash
git init
```

然后：

```bash
git add .
git commit -m "Day 1: initialize Spring Boot stock backend"
```

连接 GitHub：

```bash
git remote add origin https://github.com/你的用户名/stock-ai-platform.git
```

然后：

```bash
git branch -M main
git push -u origin main
```

---

# Day 1 验收标准

今天完成下面 **8 项**：

* [ ] AWS Account
* [ ] IAM 基础概念
* [ ] Java 21
* [ ] IntelliJ IDEA
* [ ] Git
* [ ] AWS CLI
* [ ] Spring Boot 3
* [ ] `/api/health` 成功运行

最终：

```text
Browser
   ↓
localhost:8080
   ↓
Spring Boot
   ↓
HealthController
   ↓
Stock AI Platform is running!
```

**Day 2** 我们开始真正搭股票系统：**PostgreSQL + Spring Data JPA + Docker + AWS RDS**，并建立 `users / stocks / portfolios / orders / positions` 数据库结构。
