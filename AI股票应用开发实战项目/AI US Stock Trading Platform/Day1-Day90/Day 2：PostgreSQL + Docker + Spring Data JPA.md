# Day 2：PostgreSQL + Docker + Spring Data JPA

今天目标：

> **把股票 App 的数据库层搭起来，并让 Spring Boot 成功连接 PostgreSQL。**

今天完成后：

```text
Spring Boot
     │
     ▼
PostgreSQL
     │
     ├── stocks
     ├── users
     ├── portfolios
     ├── positions
     └── orders
```

---

## 1. 安装 Docker

如果还没有安装：

[Docker Desktop 官方](https://www.docker.com/products/docker-desktop/?utm_source=chatgpt.com)

检查：

```bash
docker --version
docker compose version
```

两个命令都应该有版本号。

---

# 2. 创建 PostgreSQL Docker

在项目根目录：

```text
stock-ai-platform/
└── stock-backend/
```

创建：

```text
docker-compose.yml
```

内容：

```yaml
services:

  postgres:
    image: postgres:16
    container_name: stock-postgres
    environment:
      POSTGRES_DB: stockdb
      POSTGRES_USER: stockuser
      POSTGRES_PASSWORD: stockpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

启动：

```bash
docker compose up -d
```

检查：

```bash
docker ps
```

应该看到：

```text
stock-postgres
```

---

# 3. PostgreSQL 连接参数

现在我们的 Spring Boot 使用：

```text
Host: localhost
Port: 5432
Database: stockdb
Username: stockuser
Password: stockpassword
```

连接字符串：

```text
jdbc:postgresql://localhost:5432/stockdb
```

---

# 4. 加入 Spring Data JPA

打开：

```text
pom.xml
```

增加：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

然后：

```bash
./mvnw clean install
```

---

# 5. 配置 PostgreSQL

创建：

```text
src/main/resources/application.yml
```

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/stockdb
    username: stockuser
    password: stockpassword
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true

server:
  port: 8080
```

现在启动：

```bash
./mvnw spring-boot:run
```

如果日志出现：

```text
HikariPool
PostgreSQL
Started StockBackendApplication
```

说明连接成功。

---

# 6. 创建第一个 Entity：Stock

目录：

```text
src/main/java/com/stockai/entity/
```

创建：

```text
Stock.java
```

```java
package com.stockai.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "stocks")
public class Stock {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String symbol;

    private String name;

    private String exchange;

    private String sector;

    public Long getId() {
        return id;
    }

    public String getSymbol() {
        return symbol;
    }

    public void setSymbol(String symbol) {
        this.symbol = symbol;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getExchange() {
        return exchange;
    }

    public void setExchange(String exchange) {
        this.exchange = exchange;
    }

    public String getSector() {
        return sector;
    }

    public void setSector(String sector) {
        this.sector = sector;
    }
}
```

---

# 7. 创建 Repository

目录：

```text
repository/
```

创建：

```text
StockRepository.java
```

```java
package com.stockai.repository;

import com.stockai.entity.Stock;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface StockRepository
        extends JpaRepository<Stock, Long> {

    Optional<Stock> findBySymbol(String symbol);
}
```

这就是 Spring Data JPA 的核心。

以后：

```java
stockRepository.findBySymbol("AAPL");
```

就可以查询：

```text
AAPL
```

---

# 8. 创建 Stock Service

```java
package com.stockai.service;

import com.stockai.entity.Stock;
import com.stockai.repository.StockRepository;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class StockService {

    private final StockRepository repository;

    public StockService(StockRepository repository) {
        this.repository = repository;
    }

    public List<Stock> findAll() {
        return repository.findAll();
    }

    public Stock save(Stock stock) {
        return repository.save(stock);
    }
}
```

---

# 9. 创建 Stock API

```java
package com.stockai.controller;

import com.stockai.entity.Stock;
import com.stockai.service.StockService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/stocks")
public class StockController {

    private final StockService service;

    public StockController(StockService service) {
        this.service = service;
    }

    @GetMapping
    public List<Stock> getStocks() {
        return service.findAll();
    }

    @PostMapping
    public Stock createStock(@RequestBody Stock stock) {
        return service.save(stock);
    }
}
```

---

# 10. 测试 API

启动：

```bash
./mvnw spring-boot:run
```

打开：

```text
http://localhost:8080/api/stocks
```

第一次应该：

```json
[]
```

---

## 11. 添加 AAPL

使用 Postman、IntelliJ HTTP Client 或 curl：

```bash
curl -X POST http://localhost:8080/api/stocks \
-H "Content-Type: application/json" \
-d '{
  "symbol": "AAPL",
  "name": "Apple Inc.",
  "exchange": "NASDAQ",
  "sector": "Technology"
}'
```

然后：

```bash
curl http://localhost:8080/api/stocks
```

应该得到类似：

```json
[
  {
    "id": 1,
    "symbol": "AAPL",
    "name": "Apple Inc.",
    "exchange": "NASDAQ",
    "sector": "Technology"
  }
]
```

---

# 12. 再加入 4 支股票

```text
AAPL
NVDA
TSLA
AMD
MSFT
```

这五支股票以后会成为我们的测试数据。

---

# 13. 今天理解一个重要概念

现在：

```text
StockController
       ↓
StockService
       ↓
StockRepository
       ↓
PostgreSQL
```

这是标准的：

**Controller → Service → Repository → Database**

以后交易系统也是一样：

```text
OrderController
       ↓
OrderService
       ↓
OrderRepository
       ↓
PostgreSQL
```

---

# 14. 今天先不要上 AWS RDS

这一点非常重要。

现在：

```text
开发环境
   ↓
Docker PostgreSQL
   ↓
localhost
```

等数据库结构稳定以后：

```text
Production
   ↓
AWS RDS PostgreSQL
```

这样可以避免你一开始就产生 AWS 数据库费用。

---

# Day 2 验收

今天完成：

* [ ] Docker Desktop
* [ ] PostgreSQL 16
* [ ] Docker Compose
* [ ] Spring Data JPA
* [ ] PostgreSQL connection
* [ ] Stock Entity
* [ ] Stock Repository
* [ ] Stock Service
* [ ] Stock REST API
* [ ] 成功保存 AAPL
* [ ] 成功查询 AAPL

最终架构：

```text
              Stock App
                  │
                  ▼
          Spring Boot 3
                  │
          ┌───────┴───────┐
          ▼               ▼
    StockController    StockService
                          │
                          ▼
                  StockRepository
                          │
                          ▼
                    PostgreSQL
                       Docker
```

**Day 3：我们开始接入 Alpaca，做真正的美国股票实时行情 API。**
届时会实现：

```text
GET /api/stocks/AAPL/quote
```

返回 **AAPL 当前 Bid / Ask / Price / Volume**，这才正式进入股票应用开发。
