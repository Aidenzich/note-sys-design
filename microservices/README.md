# Microservices Architecture (微服務架構)

微服務架構將單體應用 (Monolith) 拆分為一組小型、獨立部署的服務，以提高開發速度和可擴展性。

## 1. 核心原則

| 原則 | 說明 |
|:---|:---|
| **單一職責** | 每個服務只做一件事 (Loose Coupling, High Cohesion) |
| **獨立部署** | 服務可獨立更新、回滾，不影響其他服務 |
| **數據封裝** | 每個服務擁有自己的資料庫，不共享 DB |
| **分散式治理** | 團隊可選用不同的技術堆疊 (Polyglot) |
| **自動化** | 必須依賴 CI/CD 和自動化測試 |

---

## 2. 通訊模式 (Communication)

### 2.1 同步 (Synchronous)

Client 等待 Server 回應。

- **REST (HTTP/JSON)**: 最通用，但性能和型別安全性較弱
- **gRPC (Protobuf)**: 高效、強型別，適合內部服務通訊
- **GraphQL**: 適合前端聚合多個服務的資料 (BFF Pattern)

**缺點**：
- 請求鏈路長時延遲增加
- 級聯故障 (Cascading Failure) 風險高

### 2.2 異步 (Asynchronous)

Client 發送訊息後不等待，由 Broker 傳遞。

- **Message Queue** (RabbitMQ/SQS): 點對點，任務分發
- **Event Bus** (Kafka): 發布/訂閱，事件驅動架構 (EDA)

**優點**：
- 解耦 (Decoupling)
- 削峰填谷 (Traffic Shaping)
- 提高可用性

---

## 3. 服務發現 (Service Discovery)

在動態環境中 (Kubernetes)，服務實例的 IP 是變動的。

### 3.1 Client-side Discovery

Client 查詢 Service Registry 獲取可用實例列表，自行負載均衡。

```
Client ──Query──→ Service Registry (Eureka/Consul)
   │
   └──Call──→ Service Instance (10.0.0.5)
```
- ✅ 少一層跳轉，延遲低
- ❌ Client 需實作負載均衡邏輯

### 3.2 Server-side Discovery

Client 呼叫 Load Balancer (DNS/VIP)，由 LB 查詢 Registry 並轉發。

```
Client ──Call──→ Load Balancer (Nginx/K8s Service)
                       │
                       └──Route──→ Service Instance
```
- ✅ Client 簡單，對語言無關
- ❌ 多一層跳轉

---

## 4. API Gateway Pattern

作為微服務的統一入口。

### 4.1 核心功能

1. **路由 (Routing)**: `/users` → User Service, `/orders` → Order Service
2. **聚合 (Aggregation)**: 調用多個服務並合併結果 (BFF)
3. **安全 (Security)**: 認證 (AuthN)、SSL Termination
4. **限流 (Rate Limiting)**: 保護後端服務
5. **協議轉換**: HTTP → gRPC

### 4.2 Backend for Frontend (BFF)

為不同客戶端 (Web/Mobile/IoT) 建立專屬的 Gateway/Adapter，提供裁剪後的數據。

```
Mobile App ──→ Mobile BFF ──→ Microservices
Web App    ──→ Web BFF    ──→ Microservices
```

---

## 5. 資料一致性

*(詳見 [Distributed Transactions](../reliability/distributed-tx/README.md))*

- **Database per Service**: 嚴格禁止跨庫 Join
- **Saga Pattern**: 管理長事務
- **CQRS (Command Query Responsibility Segregation)**: 讀寫分離，解決複雜查詢問題

---

## 6. SRE 與可觀測性 (Observability)

微服務讓除錯變得極其困難，必須依賴三大支柱：

### 6.1 Logging (ELK/EFK)
- 集中收集日誌
- **Correlation ID**: 在所有日誌中注入 Request ID，串聯整個調用鏈

### 6.2 Metrics (Prometheus/Grafana)
- 監控 RED 指標 (Rate, Errors, Duration)
- 業務指標 (訂單量、活躍用戶)
- 基礎設施 (CPU/RAM)

### 6.3 Tracing (Jaeger/Zipkin)
- 分散式追蹤，視覺化請求在各服務間的流轉
- 分析延遲瓶頸

---

## 7. 遷移策略 (Strangler Fig Pattern)

如何從單體 (Monolith) 遷移到微服務？**絞殺榕模式**。

1. 在 Monolith 前面加 Proxy/Gateway
2. 識別一個邊緣功能 (如：Email Service)
3. 開發新的微服務
4. Gateway 將流量切換到新服務
5. 重複步驟，直到 Monolith 消失 (或只剩核心)

**不要一次重寫 (Big Bang Rewrite)！必死無疑。**

---

## 8. Anti-Patterns (微服務陷阱)

| 陷阱 | 說明 |
|:---|:---|
| **Distributed Monolith** | 服務間強耦合，必須同時部署 |
| **Shared Database** | 多個服務讀寫同一個 DB，毫無隔離 |
| **Microservices Envy** | 團隊太小 (2-3人) 卻硬要拆分 10 個服務 |
| **Nanoservices** | 拆分太細 (如每個 Function 一個服務)，通訊開銷大於業務邏輯 |

---

## 9. Reference
- [Microservices Patterns (Chris Richardson)](https://microservices.io/)
- [Building Microservices (Sam Newman)](https://samnewman.io/books/building_microservices/)
