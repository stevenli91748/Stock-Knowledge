# Day 3：接入 Alpaca 美股行情 API

今天目标非常明确：

> **让你的 Spring Boot 后端从 Alpaca 获取真实美股行情。**

完成后：

```text
React
  ↓
Spring Boot
  ↓
Alpaca Market Data API
  ↓
AAPL / NVDA / TSLA
```

---

## 1. 注册 Alpaca

进入：

[Alpaca 官方网站](https://alpaca.markets/?utm_source=chatgpt.com)

注册账户后进入 **Paper Trading**。

我们今天只使用：

**Paper Trading / Market Data**

不要连接真实交易账户。

Alpaca 官方 API 文档：

[Alpaca API Documentation](https://docs.alpaca.markets/?utm_source=chatgpt.com)

---

# 2. 获取 API Key

在 Alpaca Dashboard 找到：

```text
Paper Trading
→ API Keys
```

得到：

```text
API_KEY
SECRET_KEY
```

**不要把这两个 Key 放进 GitHub。**

---

# 3. 不要直接写死 API Key

错误：

```java
String apiKey = "xxxxxxxx";
```

正确：

```text
Environment Variables
        ↓
Spring Boot
        ↓
Alpaca
```

Mac/Linux：

```bash
export ALPACA_API_KEY="你的key"
export ALPACA_SECRET_KEY="你的secret"
```

检查：

```bash
echo $ALPACA_API_KEY
```

Windows PowerShell：

```powershell
$env:ALPACA_API_KEY="你的key"
$env:ALPACA_SECRET_KEY="你的secret"
```

---

# 4. Spring Boot 配置

`application.yml`：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/stockdb
    username: stockuser
    password: stockpassword

alpaca:
  api-key: ${ALPACA_API_KEY}
  secret-key: ${ALPACA_SECRET_KEY}
  data-url: https://data.alpaca.markets
  trading-url: https://paper-api.alpaca.markets
```

这里要理解：

```text
data-url
    ↓
股票行情

trading-url
    ↓
Paper Trading
```

今天主要使用 **data-url**。

---

# 5. 创建 AlpacaClient

目录：

```text
client/
└── AlpacaClient.java
```

代码：

```java
package com.stockai.client;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class AlpacaClient {

    private final RestClient restClient;

    private final String apiKey;
    private final String secretKey;

    public AlpacaClient(
            @Value("${alpaca.api-key}") String apiKey,
            @Value("${alpaca.secret-key}") String secretKey,
            @Value("${alpaca.data-url}") String dataUrl) {

        this.apiKey = apiKey;
        this.secretKey = secretKey;

        this.restClient = RestClient.builder()
                .baseUrl(dataUrl)
                .defaultHeader("APCA-API-KEY-ID", apiKey)
                .defaultHeader("APCA-API-SECRET-KEY", secretKey)
                .build();
    }
}
```

---

# 6. 获取最新 Quote

我们先实现：

```text
GET /v2/stocks/AAPL/quotes/latest
```

在 `AlpacaClient` 增加：

```java
public String getLatestQuote(String symbol) {

    return restClient.get()
            .uri(uriBuilder ->
                    uriBuilder
                            .path("/v2/stocks/{symbol}/quotes/latest")
                            .build(symbol))
            .retrieve()
            .body(String.class);
}
```

---

# 7. 创建 Stock Service

修改：

```text
StockService.java
```

加入：

```java
private final AlpacaClient alpacaClient;

public StockService(
        StockRepository repository,
        AlpacaClient alpacaClient) {

    this.repository = repository;
    this.alpacaClient = alpacaClient;
}
```

然后：

```java
public String getLatestQuote(String symbol) {

    return alpacaClient.getLatestQuote(symbol);
}
```

---

# 8. 创建 Quote API

`StockController.java`：

```java
@GetMapping("/{symbol}/quote")
public String getQuote(@PathVariable String symbol) {

    return service.getLatestQuote(symbol);
}
```

现在你的 API：

```text
GET /api/stocks/AAPL/quote
```

---

# 9. 启动

先确认 Docker PostgreSQL：

```bash
docker ps
```

然后：

```bash
./mvnw spring-boot:run
```

---

# 10. 测试 AAPL

浏览器：

```text
http://localhost:8080/api/stocks/AAPL/quote
```

或者：

```bash
curl http://localhost:8080/api/stocks/AAPL/quote
```

正常情况下会返回类似：

```json
{
  "quote": {
    "ap": 230.12,
    "as": 100,
    "bp": 230.10,
    "bs": 200
  }
}
```

其中：

```text
ap = Ask Price
as = Ask Size

bp = Bid Price
bs = Bid Size
```

**注意：具体价格会随市场实时变化。**

---

# 11. 改成真正的 Java DTO

今天先能跑起来。

下一步不要长期使用：

```java
String
```

而应该：

```text
JSON
 ↓
Java DTO
 ↓
StockQuote
```

创建：

```text
dto/StockQuote.java
```

例如：

```java
package com.stockai.dto;

public record StockQuote(
        String symbol,
        double askPrice,
        double bidPrice,
        long askSize,
        long bidSize
) {
}
```

以后我们会进一步完善成：

```text
StockQuote
├── symbol
├── bidPrice
├── askPrice
├── bidSize
├── askSize
├── timestamp
└── conditions
```

---

# 12. 今天非常重要：行情和交易是两个系统

你现在连接的是：

```text
Alpaca Market Data
```

它负责：

```text
AAPL price
NVDA price
TSLA price
Volume
Quotes
Trades
Bars
```

而交易：

```text
BUY
SELL
ORDER
POSITION
ACCOUNT
```

走：

```text
https://paper-api.alpaca.markets
```

所以我们的系统会分成：

```text
                 Alpaca
                   │
          ┌────────┴────────┐
          ▼                 ▼
    Market Data          Trading
          │                 │
          ▼                 ▼
    StockService        OrderService
```

这个架构非常重要。

---

# 13. 今天测试 5 支股票

依次访问：

```text
/api/stocks/AAPL/quote
/api/stocks/NVDA/quote
/api/stocks/TSLA/quote
/api/stocks/AMD/quote
/api/stocks/MSFT/quote
```

如果都能返回数据：

**Day 3 成功。**

---

# 14. Git 提交

```bash
git add .
git commit -m "Day 3: integrate Alpaca market data API"
git push
```

---

# Day 3 最终架构

```text
                 Internet
                    │
                    ▼
              Spring Boot
                    │
             StockController
                    │
              StockService
                    │
              AlpacaClient
                    │
                    ▼
          Alpaca Market Data API
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
       AAPL        NVDA        TSLA
        │           │           │
        └───────────┼───────────┘
                    ▼
                 Quote
```

### 今天你真正学到的

**Java + Spring Boot + REST Client + API Authentication + Alpaca Market Data**

下一步 **Day 4**：

> **实时行情 + WebSocket + K线数据 + Redis**

我们会把现在的：

```text
GET /AAPL/quote
```

升级成：

```text
实时行情
     ↓
WebSocket
     ↓
Redis
     ↓
Spring Boot
     ↓
前端实时刷新
```

这一步开始，你的项目就真正像一个**美国股票实时交易平台**了。
