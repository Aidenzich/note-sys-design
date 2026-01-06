# DynamoDB 深度解析

Amazon DynamoDB 是一個完全託管的 NoSQL 資料庫，提供單位數毫秒的延遲，適合極大規模的應用。

## 1. 核心架構

### 1.1 Hash + Range Key

DynamoDB 表必須定義 Primary Key，有兩種模式：

| 模式 | 組成 | 說明 | 查詢能力 |
|:---|:---|:---|:---|
| **Simple PK** | Partition Key (PK) | 僅有 PK，PK 必須唯一 | 只能用 GetItem 查一筆 |
| **Composite PK** | Partition Key (PK) + Sort Key (SK) | 組合必須唯一，PK 用於分片，SK 用於排序 | 支援 Query 範圍查詢 |

**範例：訂單表**
- PK: `UserID` (將同一用戶的資料分到同分區)
- SK: `OrderDate` (最新的訂單排前面)

### 1.2 資料分區 (Partitioning)

DynamoDB 基於 Partition Key 的 Hash 值將資料分散到多個 **Partition**。

```
Partition Key Value ──Hash──→ [ Hash Range A ] ──→ Partition 1
                              [ Hash Range B ] ──→ Partition 2
                              [ Hash Range C ] ──→ Partition 3
```

**Adaptive Capacity:**
如果某個 Partition 變熱（Hot Partition），DynamoDB 會自動隔離該 Partition 並增加其吞吐量上限。

---

## 2. 索引機制

### 2.1 Local Secondary Index (LSI)

- **範圍**：必須與 Table 相同的 PK，但不同的 SK
- **限制**：**必須在創建 Table 時定義**，不可事後修改
- **儲存**：與原始資料在同一個 Partition (Item Collection)
- **大小限制**：每個 Item Collection (相同的 PK) 總大小不能超過 10GB
- **一致性**：支援強一致性 (Strongly Consistent)

### 2.2 Global Secondary Index (GSI)

- **範圍**：可以是完全不同的 PK 和 SK
- **限制**：隨時可以創建/刪除
- **儲存**：實際上是另一張隱藏的 Table (資料會非同步複製過去)
- **一致性**：**僅支援最終一致性** (Eventual Consistency)
- **吞吐量**：擁有獨立的 RCU/WCU 設定

**比較總結**

| 特性 | LSI | GSI |
|:---|:---|:---|
| **PK** | 必須與 Base Table 相同 | 可不同 |
| **SK** | 必須不同 | 可不同 |
| **創建時機** | 僅建表時 | 隨時 |
| **一致性** | 強一致性 / 最終一致性 | 僅最終一致性 |
| **大小限制** | 10GB per Partition Key | 無限制 |
| **成本** | 共享 Base Table 吞吐量 | 獨立吞吐量 cost |

---

## 3. 一致性與事務

### 3.1 讀取一致性

| 模式 | 說明 | 成本 |
|:---|:---|:---|
| **Eventual Consistent** (預設) | 可能讀到舊資料 (複製延遲通常 < 1s) | 0.5 RCU / 4KB |
| **Strongly Consistent** | 保證讀到最新寫入 | 1.0 RCU / 4KB |

### 3.2 DynamoDB Transactions (ACID)

AWS 支援跨多個 Item 甚至是跨 Table 的 ACID 事務。

- `TransactWriteItems`: 原子寫入最多 100 個 Item
- `TransactGetItems`: 原子讀取最多 100 個 Item
- **成本**：標準操作的 2 倍 (因為需 Two-Phase Commit)

---

## 4. 吞吐量管理

### 4.1 RCU / WCU

- **Read Capacity Unit (RCU)**: 每秒讀取 1 個 4KB Item (強一致) 或 2 個 Item (最終一致)
- **Write Capacity Unit (WCU)**: 每秒寫入 1 個 1KB Item

### 4.2 On-Demand vs Provisioned

| 模式 | 說明 | 適用場景 |
|:---|:---|:---|
| **Provisioned** | 預先設定 RCU/WCU，可設 Auto Scaling | 流量穩定、可預測 |
| **On-Demand** | 按實際請求付費，自動處理突發 | 流量難預測、新產品 |

---

## 5. DAX (DynamoDB Accelerator)

DAX 是 DynamoDB 專用的 In-Memory Write-Through Cache。

```
App ──→ DAX Cluster ──→ DynamoDB
```

- **優點**：
  - 讀取延遲從毫秒級 (ms) 降至微秒級 (μs)
  - 減少 DynamoDB 的 RCU 消耗
  - API 相容，不需改程式碼 (只需換 Client)
- **缺點**：
  - 額外成本
  - 僅支援最終一致性讀取

---

## 6. Stream 與架構模式

### 6.1 DynamoDB Streams

記錄 Table 的變更日誌 (Insert, Update, Delete)，保留 24 小時。

**常見應用：**
1. **觸發 Lambda**：當訂單建立時，觸發 Lambda 發送 Email
2. **複製到 ES**：將變更同步到 Elasticsearch 做全文檢索
3. **跨 Region 複製**：Global Tables 的底層機制

### 6.2 Global Tables

- **主動-主動 (Active-Active)** 多區域複製
- 基於 DynamoDB Streams 自動同步
- **最後寫入者勝 (Last Writer Wins)** 解決衝突

---

## 7. 設計模式 (Best Practices)

### 7.1 單表設計 (Single Table Design)

利用 Overloading Key 將不同實體存在同一張表，減少 Join 需求。

| PK | SK | Data | 說明 |
|:---|:---|:---|:---|
| `USER#123` | `PROFILE` | {name: "John"} | 用戶資料 |
| `USER#123` | `ORDER#1001` | {total: 99} | 用戶訂單 |
| `ORDER#1001`| `ITEM#A` | {count: 1} | 訂單項目 |

- Query `PK=USER#123`：一次取出 Profile + 所有 Orders

### 7.2 處理熱點 Key (Hot Partition)

- **Write Sharding**: 在 PK 後面加隨機後綴 (`Key#1`, `Key#2`...)
- **Read Sharding**: 讀取時平行 Query 所有後綴

---

## 8. Reference
- [AWS DynamoDB Guide](https://docs.aws.amazon.com/dynamodb/)
- [The DynamoDB Book (Alex DeBrie)](https://www.dynamodbbook.com/)
