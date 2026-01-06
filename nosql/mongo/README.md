# MongoDB 深度解析

MongoDB 是最流行的文件導向 (Document-Oriented) NoSQL 資料庫，使用 BSON (Binary JSON) 格式儲存資料，兼具靈活性與強大的查詢能力。

## 1. 核心架構

### 1.1 Replica Set (複本集) - 高可用

MongoDB 的基礎部署單元是 Replica Set，通常由 3 個節點組成：

- **Primary**: 唯一的寫入節點，讀取預設也在這裡
- **Secondary**: 透過 Oplog (Operation Log) 異步複製 Primary 的變更
- **Arbiter** (Optional): 僅參與投票，不存資料

**自動故障轉移 (Failover):**
當 Primary 當機，Secondaries 會觸發選舉，選出新的 Primary (基於優先級和數據新鮮度)。

### 1.2 Sharding (分片) - 水平擴展

當單機容量不足時，使用 Sharded Cluster。

| 組件 | 功能 |
|:---|:---|
| **Mongos** (Router) | 應用程式接入點，負責將查詢路由到正確的 Shard |
| **Config Server** | 儲存 Metadata (Chunks 分佈、路由表) |
| **Shard** | 實際儲存資料的 Replica Set |

**Chunk Migration:**
Balancer 會自動在 Shards 之間遷移 Chunks，保持數據均衡。

---

## 2. 儲存引擎 (WiredTiger)

### 2.1 核心特性

- **Document Level Concurrency**: 寫入鎖僅限於 Document 級別 (舊版 MMAPv1 是 Collection 級別)
- **Compression**: 預設啟用 Snappy 壓縮，節省空間
- **Checkpointing**: 每 60 秒將變更 flush 到磁碟 (Data Files)，在此之前依賴 Journal 保證 Durability

### 2.2 Journaling (WAL)

類似 RDB 的 WAL，所有寫入先寫入 Journal (預設 100ms 刷盤)，再寫入記憶體。確保 Crash Recovery。

---

## 3. 一致性與讀寫關注

### 3.1 Write Concern (寫入關注)

控制寫入確認的層級：

| Level | 說明 | 安全性 | 速度 |
|:---|:---|:---|:---|
| `w: 1` | Primary 確認寫入記憶體即返回 (預設) | 普通 | 快 |
| `w: "majority"` | 大多數節點確認寫入 (避免 Rollback) | 高 | 中 |
| `j: true` | 確認寫入 Journal (磁碟) | 最高 | 慢 |

### 3.2 Read Preference (讀取偏好)

控制讀取從哪個節點獲取：

| 模式 | 說明 | 適用場景 |
|:---|:---|:---|
| `primary` | 只讀 Primary (預設) | 強一致性 |
| `primaryPreferred` | 優先讀 Primary，掛了讀 Secondary | 高可用 |
| `secondary` | 只讀 Secondary | 讀寫分離 (可接受延遲) |
| `secondaryPreferred` | 優先 Secondary | 分析型查詢 |
| `nearest` | 讀延遲最低的節點 | 全球分佈優化 |

### 3.3 Read Concern

- `local`: 讀取節點當前數據 (可能被 Rollback)
- `majority`: 讀取已被大多數節點確認的數據 (保證不 Rollback)
- `linearizable`: 線性一致性 (類似 RDBMS)

---

## 4. Sharding 策略

選擇合適的 Shard Key 至關重要。

### 4.1 Ranged Sharding

基於 Shard Key 的範圍劃分 Chunks。
- **優點**: 支援範圍查詢高效 (`age > 20`)
- **缺點**: 連續寫入可能導致熱點 (如 Timestamp)

### 4.2 Hashed Sharding

基於 Shard Key 的 Hash 值劃分。
- **優點**: 數據分佈極其均勻
- **缺點**: 範圍查詢效率差 (需廣播到所有 Shards)

### 4.3 Zone Sharding (Data Locality)

將特定範圍的數據綁定到特定的 Shard (如：US 用戶存 US Shard)。

---

## 5. 索引與查詢優化

### 5.1 索引類型

- **Single Field**: `createIndex({score: 1})`
- **Compound**: `createIndex({username: 1, date: -1})` (順序很重要!)
- **Multikey**: 索引陣列欄位 (如 Tags)
- **Text**: 全文檢索
- **Geospatial**: `2dsphere` 地理位置查詢

### 5.2 ESR Rule (Compound Index)

設計複合索引的最佳實踐順序：

1. **E (Equality)**: 精確匹配的欄位放最前 (`status: "active"`)
2. **S (Sort)**: 排序欄位放中間 (`sort: {date: 1}`)
3. **R (Range)**: 範圍查詢欄位放最後 (`age > 18`)

```javascript
// Query: find({status: "A", age: {$gt: 20}}).sort({date: 1})
// Index: {status: 1, date: 1, age: 1}  ✅ Fits ESR
```

---

## 6. Schema Design Patterns

### 6.1 Embedding vs Referencing

| 模式 | 範例 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **Embedding** (內嵌) | User doc 包含 Address doc | 單次讀取獲取所有數據 (原子性) | Document 大小限制 (16MB) |
| **Referencing** (引用) | User 存此 `address_id` | 數據正規化，無大小限制 | 需要 `$lookup` (Join) 較慢 |

**決策法則:**
- **One-to-Few**: Embedding (如：每人幾個地址)
- **One-to-Many**: Embedding (如：每篇幾個評論)
- **One-to-Squillions**: Referencing (如：系統日誌)

### 6.2 Bucket Pattern (時序優化)

將大量小文檔 (如每分鐘的感測器讀數) 合併為一個文檔 (Bucket)。

```json
{
  "sensor_id": 123,
  "date": "2024-01-01",
  "readings": [
    {"ts": 1001, "val": 25},
    {"ts": 1002, "val": 26},
    ...
  ]
}
```
- 減少索引大小
- 提高讀取效率

---

## 7. Reference
- [MongoDB Architecture Guide](https://www.mongodb.com/docs/manual/core/architecture-introduction/)
- [Data Modeling Introduction](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/)