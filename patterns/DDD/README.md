# 《DDD 領域驅動設計：從戰略到戰術的實務筆記》

## 壹、DDD 的核心精神

DDD（Domain-Driven Design）不是框架，而是一套把「業務知識」放在軟體中心的設計方法。

- 用**統一語言（Ubiquitous Language）**讓 PM、RD、QA 說同一種話。
- 用**邊界上下文（Bounded Context）**切分模型，避免同一名詞在不同子系統意義混亂。
- 用**聚合（Aggregate）與封裝**保護業務不變條件（Invariant）。
- 用**分層架構**隔離技術細節，讓資料庫、MQ、框架可替換。

一句話總結：DDD 解的是「業務複雜度」，不是「所有系統都物件化」。

## 貳、戰略設計（Strategic Design）

### 1. 子域切分（Subdomain）

- **Core Domain（核心域）：** 直接形成競爭優勢，最值得投資設計能力。
- **Supporting Subdomain（支撐域）：** 重要但非核心差異化能力。
- **Generic Subdomain（通用域）：** 可買現成方案或直接採通用實作。

### 2. 邊界上下文（Bounded Context）

同一詞在不同 Context 可有不同模型，例如 `Order` 在「訂單」與「物流」上下文的欄位與規則可不同，不應共用同一個巨型物件。

### 3. 上下文關係（Context Map）

常見整合關係：

- **Customer/Supplier：** 上游模型驅動下游整合節奏。
- **Conformist：** 下游直接服從上游模型。
- **ACL（Anti-Corruption Layer）：** 用轉換層保護本地模型，不被外部模型污染。
- **Published Language / Open Host Service：** 以公開契約穩定整合。

## 參、戰術設計（Tactical Design）

DDD 在程式碼層面反對貧血模型（只有 getter/setter），主張「資料 + 行為 + 規則」同時封裝。

### 1. Entity（實體）

- 有識別（ID）且生命周期中會變化。
- 判等通常依據 ID，而非全欄位值。

### 2. Value Object（值物件）

- 無識別，值相等即物件相等。
- 應保持不可變（Immutable），如 `Money`、`Address`。

### 3. Aggregate / Aggregate Root（聚合 / 聚合根）

- 聚合是一次一致性邊界。
- 聚合根是外界唯一入口，外界不直接修改內部 Entity。
- 不變條件在聚合根內被保證，例如「已付款訂單不可取消」。

### 4. Domain Service（領域服務）

當規則不自然屬於某個 Entity/VO，且仍屬領域邏輯時，放在 Domain Service。

### 5. Application Service（應用服務）

負責流程協調（交易、呼叫 Repository、發送事件），不承載核心業務規則。

### 6. Repository（儲存庫）

Repository 是「領域層看待資料的集合介面」，不是 ORM 的別名。實務守則：

- 以**聚合根**為主要讀寫單位（可有查詢模型例外）。
- 負責持久化與重建聚合，隱藏儲存細節。
- 介面定義放在 Domain，實作放在 Infrastructure。

## 肆、Outbox Pattern（發件箱模式）與一致性

Outbox 用來解決 Dual Write（DB 成功但 MQ 失敗）：

1. 應用服務開啟交易。
2. 透過 Repository 寫入業務資料。
3. 同交易寫入 `outbox` 事件紀錄。
4. 提交交易，確保狀態與事件同生共死。
5. 背景派送器（Poller/CDC）把 Outbox 事件送到 Kafka/SQS。

關鍵觀念：

- Outbox 保證「至少一次（at-least-once）」投遞常見，消費者要做冪等。
- 事件需帶 `event_id`、`aggregate_id`、`occurred_at` 等欄位便於追蹤與去重。

## 伍、跨 Context 流程：Order -> Inventory（Saga）

跨資料庫時不要期待本地交易 rollback 全世界，通常改用 Saga 補償。

### Happy Path (監聽與處理流)

從訂單 (Order) 到庫存 (Inventory) 的完整流程可拆解為：**「發送 -> 訂閱 -> 重建 -> 處理 -> 持久化」**。

**資料流路徑：**
`Order 系統 (Outbox)` -> `Kafka (Topic: order-events)` -> **`Inventory Consumer`** -> **`Inventory EventHandler`** -> **`Inventory Repository`** -> **`Inventory Aggregate`** -> **`Inventory Repository (Save)`**

#### 庫存端 (Inventory) 的詳細步驟：

1.  **訂閱與接收 (Subscribe & Consume)**：背景 Kafka Consumer 盯著 `order-events` 主題，收到 `OrderCreatedEvent` 時將 JSON 轉為物件（如 Dataclass）。
2.  **派發給事件處理器 (Dispatch)**：Consumer 將事件轉交給應用服務層的 `InventoryEventHandler`。
3.  **透過 Repository 重建聚合根 (Reconstruct)**：處理器呼叫 `repo.get(product_id)`，從資料庫撈出資料並實體化為活生生的 `Inventory` 聚合根。**防腐關鍵：不直接寫 SQL UPDATE。**
4.  **聚合根執行業務邏輯 (Execute)**：呼叫 `inventory.reserve(quantity)`。聚合根在記憶體中檢查規則（如：剩餘量是否足夠）並更新自身狀態。
5.  **透過 Repository 存檔 (Save)**：處理器將狀態改變後的聚合根交還給 `repo.save(inventory)`，由 Repository 負責 SQL 交易與持久化。

#### Python 偽代碼實作範例

```python
# 這是庫存服務的應用層 (Application Service)
class InventoryEventHandler:
    def __init__(self, repository: InventoryRepository):
        self.repo = repository

    # 當 Kafka Consumer 收到訊息時，會呼叫這個方法
    def handle_order_created(self, event: OrderCreatedEvent):
        # 3. 重建聚合根 (Reconstruct)
        inventory_aggregate = self.repo.get(event.product_id)

        if not inventory_aggregate:
            raise ValueError("找不到該商品庫存")

        # 4. 執行業務邏輯 (由聚合根自己把關)
        inventory_aggregate.reserve(event.quantity)

        # 5. 存檔持久化 (Save)
        self.repo.save(inventory_aggregate)
```

### 補償路徑

1. 下游失敗發布 `ReservationFailed` / `PaymentFailed`。
2. `Order` Context 消費後執行聚合行為（如 `cancel()`）。
3. 產生補償事件通知其他上下文回收資源。

## 陸、何時適合用 DDD

- **適合：** 規則多、名詞容易歧義、跨團隊協作頻繁的核心業務。
- **不適合：** 純 CRUD、小型一次性工具、主要瓶頸在 I/O 或演算法而非業務規則。

實務上常見做法是「局部 DDD」：只在 Core Domain 高強度使用，其他模組保持簡潔。

## 柒、DDD 重要術語中英對照表

| English                     | 中文          | 一句話定義                                 |
| --------------------------- | ------------- | ------------------------------------------ |
| Domain                      | 領域          | 問題空間與業務知識本體。                   |
| Subdomain                   | 子域          | 領域中的子問題區塊（核心/支撐/通用）。     |
| Ubiquitous Language         | 統一語言      | 團隊共享且可落在程式碼命名的業務詞彙。     |
| Bounded Context             | 邊界上下文    | 某套模型與語義生效的邊界。                 |
| Context Map                 | 上下文地圖    | 描述各 Context 關係與整合模式。            |
| Entity                      | 實體          | 有 ID 且隨時間演化的物件。                 |
| Value Object                | 值物件        | 無 ID、以值相等、通常不可變。              |
| Aggregate                   | 聚合          | 一組維護一致性的物件邊界。                 |
| Aggregate Root              | 聚合根        | 聚合對外唯一操作入口。                     |
| Repository                  | 儲存庫        | 提供聚合的讀寫抽象，隱藏持久化細節。       |
| Factory                     | 工廠          | 封裝複雜建構流程，確保物件初始合法。       |
| Domain Service              | 領域服務      | 不屬於單一實體但屬領域規則的服務。         |
| Application Service         | 應用服務      | 協調用例流程與交易邊界。                   |
| Domain Event                | 領域事件      | 描述「領域中已發生」的重要事實。           |
| Integration Event           | 整合事件      | 對外發布、供跨 Context 溝通的事件。        |
| Outbox Pattern (OutBox)     | 發件箱模式    | 以同交易寫業務資料與事件，避免雙寫不一致。 |
| Saga                        | Saga/補償交易 | 用事件編排長交易，以補償替代全域回滾。     |
| Anti-Corruption Layer (ACL) | 防腐層        | 隔離外部模型，保護本地語義。               |
| Invariant                   | 不變條件      | 在任一合法狀態都必須成立的業務規則。       |
| Specification               | 規格模式      | 可組合的業務規則判斷物件。                 |
| Domain Model                | 領域模型      | 對業務概念、規則與關係的程式化表示。       |
| Infrastructure              | 基礎設施層    | DB、MQ、外部 API、框架與技術實作。         |
