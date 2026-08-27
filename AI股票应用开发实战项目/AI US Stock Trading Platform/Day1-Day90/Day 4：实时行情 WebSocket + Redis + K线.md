# Day 4：实时行情 WebSocket + Redis + K线

今天把 Day 3 的单次查询升级成**实时行情系统**。

目标：

```text
Alpaca WebSocket
       ↓
Spring Boot
       ↓
Redis
       ↓
WebSocket
       ↓
前端
       ↓
AAPL / NVDA / TSLA 实时价格
```

> 今天仍然只做行情，不做真实交易。

---

## 1. 为什么今天要 WebSocket？

Day 3：

```text
前端 → GET /AAPL/quote → Alpaca
```

每次查询一次。

实时股票 App 不应该每秒不停：

```text
GET
GET
GET
GET
GET
```

而应该：

```text
Alpaca
   ↓
WebSocket 长连接
   ↓
实时推送
```

这样更适合实时行情。

Alpaca 官方提供股票实时 WebSocket 数据流。[Alpaca Market Data WebSocket 文档](https://docs.alpaca.markets/docs/real-time-stock-pricing-data?utm_source=chatgpt.com)

---

# 2. 今天增加 Redis

我们的架构：

```text
                Alpaca
                   │
             WebSocket
                   │
                   ▼
          Spring Boot Backend
                   │
                   ▼
                Redis
                   │
           ┌───────┴───────┐
           ▼               ▼
       REST API         WebSocket
           │               │
           └───────┬───────┘
                   ▼
                Frontend
```

Redis 保存：

```text
AAPL latest quote
NVDA latest quote
TSLA latest quote
```

---

# 3. Docker 增加 Redis

修改 `docker-compose.yml`：

```yaml id="z4j8pr"
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

  redis:
    image: redis:7
    container_name: stock-redis
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

启动：

```bash id="4c3vpi"
docker compose up -d
```

检查：

```bash id="3x7y9h"
docker ps
```

应该看到：

```text
stock-postgres
stock-redis
```

---

# 4. Spring Boot 添加 Redis

`pom.xml`：

```xml id="zwl4w0"
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

---

# 5. 配置 Redis

`application.yml`：

```yaml id="s7s0gi"
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

现在：

```text
Spring Boot
     ↓
localhost:6379
     ↓
Redis Docker
```

---

# 6. 创建 Quote 对象

创建：

```text id="j0h1zj"
dto/StockQuote.java
```

```java id="h7q8s3"
package com.stockai.dto;

import java.time.Instant;

public record StockQuote(
        String symbol,
        double bidPrice,
        double askPrice,
        long bidSize,
        long askSize,
        Instant timestamp
) {
}
```

---

# 7. Redis Service

创建：

```text id="2pk4k7"
service/QuoteCacheService.java
```

```java id="9q5x7m"
package com.stockai.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.stockai.dto.StockQuote;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

@Service
public class QuoteCacheService {

    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;

    public QuoteCacheService(
            StringRedisTemplate redis,
            ObjectMapper objectMapper) {

        this.redis = redis;
        this.objectMapper = objectMapper;
    }

    public void save(StockQuote quote) {

        try {
            String json = objectMapper.writeValueAsString(quote);

            redis.opsForValue()
                    .set("quote:" + quote.symbol(), json);

        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

    public String get(String symbol) {

        return redis.opsForValue()
                .get("quote:" + symbol);
    }
}
```

Redis 中的数据：

```text
quote:AAPL
quote:NVDA
quote:TSLA
```

---

# 8. 建立 WebSocket API

添加依赖：

```xml id="b1r5hr"
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

然后创建：

```text id="y9r8ew"
config/WebSocketConfig.java
```

```java id="u1k4fm"
package com.stockai.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig
        implements WebSocketConfigurer {

    private final QuoteWebSocketHandler handler;

    public WebSocketConfig(
            QuoteWebSocketHandler handler) {

        this.handler = handler;
    }

    @Override
    public void registerWebSocketHandlers(
            WebSocketHandlerRegistry registry) {

        registry.addHandler(handler, "/ws/quotes")
                .setAllowedOrigins("*");
    }
}
```

---

# 9. 创建 Quote WebSocket Handler

```text id="t7c1fd"
websocket/
└── QuoteWebSocketHandler.java
```

```java id="q0u1ph"
package com.stockai.websocket;

import org.springframework.stereotype.Component;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

@Component
public class QuoteWebSocketHandler
        extends TextWebSocketHandler {

    private final Set<WebSocketSession> sessions =
            new CopyOnWriteArraySet<>();

    @Override
    public void afterConnectionEstablished(
            WebSocketSession session) {

        sessions.add(session);
    }

    @Override
    public void afterConnectionClosed(
            WebSocketSession session,
            org.springframework.web.socket.CloseStatus status) {

        sessions.remove(session);
    }

    public void broadcast(String message) {

        for (WebSocketSession session : sessions) {

            try {
                session.sendMessage(
                        new TextMessage(message)
                );
            } catch (Exception ignored) {
            }
        }
    }
}
```

---

# 10. 现在理解两个 WebSocket

这里有一个非常重要的架构概念。

实际上有**两个 WebSocket**：

```text id="2bq8n5"
             Alpaca
                │
         WebSocket #1
                │
                ▼
         Spring Boot
                │
         WebSocket #2
                │
                ▼
             Browser
```

### WebSocket #1

你的服务器：

```text
Spring Boot → Alpaca
```

接收实时行情。

### WebSocket #2

你的服务器：

```text
Spring Boot → React
```

把行情推给用户。

这样 API Key 永远留在服务器。

---

# 11. 为什么 Redis 放中间？

以后可能有：

```text
1000 users
```

同时看：

```text
AAPL
NVDA
TSLA
AMD
MSFT
```

不能：

```text
1000 users
   ↓
1000 Alpaca connections
```

应该：

```text
             Alpaca
                │
        1 market stream
                │
                ▼
         Spring Boot
                │
                ▼
             Redis
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
   User 1     User 2    User 3
```

这是以后扩展成生产系统的重要基础。

---

# 12. 今天先实现 Redis 行情缓存

先不要急着完成 Alpaca WebSocket。

先测试 Redis。

启动：

```bash id="f3f5fw"
docker compose up -d
```

进入 Redis：

```bash id="b7pj2c"
docker exec -it stock-redis redis-cli
```

输入：

```text
PING
```

应该：

```text
PONG
```

退出：

```text
exit
```

---

# 13. 添加 Quote Cache API

Controller：

```java id="7k6q4k"
@GetMapping("/{symbol}/cached")
public String getCachedQuote(
        @PathVariable String symbol) {

    return quoteCacheService.get(symbol);
}
```

然后：

```text id="e8ez9p"
GET /api/stocks/AAPL/cached
```

---

# 14. 今天先用模拟 Quote 测试

例如：

```java id="0u4a0b"
StockQuote quote = new StockQuote(
        "AAPL",
        230.10,
        230.12,
        200,
        100,
        Instant.now()
);
```

保存：

```text
quoteCacheService.save(quote);
```

然后：

```text
GET /api/stocks/AAPL/cached
```

应该得到：

```json id="15k8of"
{
  "symbol": "AAPL",
  "bidPrice": 230.10,
  "askPrice": 230.12,
  "bidSize": 200,
  "askSize": 100
}
```

---

# 15. Day 4 的重点

今天不要追求大量代码。

你必须真正理解：

### REST

```text
Request
   ↓
Response
```

适合：

```text
查询股票
查询 Portfolio
查询订单
```

### WebSocket

```text
Server
  ↓
实时 Push
  ↓
Client
```

适合：

```text
实时股价
实时订单状态
实时 P&L
```

### Redis

```text
Fast Cache
```

适合：

```text
Latest Quote
Session
热点数据
```

---

# Day 4 验收标准

* [ ] Redis Docker 正常运行
* [ ] Spring Boot 连接 Redis
* [ ] `StockQuote` DTO
* [ ] Quote Cache Service
* [ ] Redis 保存 `quote:AAPL`
* [ ] Redis 读取 `quote:AAPL`
* [ ] Spring WebSocket
* [ ] `/ws/quotes`
* [ ] 理解 Alpaca WebSocket → Spring → Browser 的架构

---

## Day 5

下一步我们把**真正的 Alpaca WebSocket 接进来**：

```text
Alpaca
   │
   │ real-time trades/quotes
   ▼
Spring Boot
   │
   ├── Redis
   │
   └── WebSocket
          │
          ▼
       Browser
          │
          ▼
      AAPL $xxx.xx
```

然后开始做**K线数据（1m / 5m / 15m / 1D）**，为后面的超短线策略、VWAP、RSI、MACD 和 AI 分析打基础。
