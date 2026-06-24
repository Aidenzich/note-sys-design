# Wide Column Store

Wide Column Store 是一種以查詢模式為中心設計的 NoSQL 資料模型，代表系統包含 Cassandra、ScyllaDB、HBase 與 Bigtable。它看起來像 Table，但底層思維更接近多維度 Key-Value：

```text
(PartitionKey, ClusteringKey, ColumnName) -> Value
```

也就是說，一筆邏輯上的 Row 不是固定欄位長度的連續紀錄，而是由多個帶有排序 Key 的欄位值組合而成。這讓它很適合存放稀疏資料、時間序列、訊息紀錄與事件流。

---

## 1. 資料模型

Wide Column 的 Primary Key 通常由兩部分組成：

```sql
PRIMARY KEY ((partition_key), clustering_key)
```

- **Partition Key**：決定資料落在哪個 Partition 或節點，是水平擴展與熱點控制的核心。
- **Clustering Key**：決定同一個 Partition 內資料如何排序，常用於時間排序或範圍查詢。
- **Column Name**：欄位名稱本身也是儲存 Key 的一部分，不同 Row 可以有不同欄位。

例如使用者資料：

```text
Key_A: Name = "Alice", Age = 25
Key_B: Name = "Bob"
```

在 Wide Column 的抽象模型裡，可以理解成：

```text
"Key_A:Age"  -> 25
"Key_A:Name" -> "Alice"
"Key_B:Name" -> "Bob"
```

`Key_B` 沒有 `Age`，底層就不需要寫入 `Key_B:Age -> NULL`。這也是 Wide Column 適合稀疏資料的原因。

> 實際 Cassandra / Bigtable 的 SSTable 格式會有 partition index、row index、metadata、compression block 等結構，不是只有純文字 Key-Value 串列；但從資料模型與查詢行為來看，`RowKey + ColumnKey -> Value` 是最重要的理解方式。

---

## 2. 與 RDBMS 的物理儲存差異

假設有兩筆資料：

- `Key_A`: `Name = "Alice"`, `Age = 25`
- `Key_B`: `Name = "Bob"`，沒有 `Age`

### RDBMS: Page + Row Layout

傳統 RDBMS 例如 MySQL/InnoDB，底層通常以 B+Tree 與 Page 組織資料。Page 是固定大小的磁碟單位，InnoDB 預設 Page Size 為 16KB。Page 內部依 Table Schema 存放 Row：

```text
[Page Header]
[Row 1: Key_A, "Alice", 25]
[Row 2: Key_B, "Bob",   NULL]
```

重點是：

- Row 依照 Table Schema 組織，欄位順序與型別相對固定。
- 即使某個欄位沒有值，也需要 NULL bitmap、變長欄位 metadata 或預設值處理。
- 寫入或更新時，資料庫需要找到對應 B+Tree Page，在 Buffer Pool 修改後再 flush 回 data file。
- 若 Page 滿了，可能觸發 Page Split，造成額外隨機 I/O 與樹結構維護成本。

### Wide Column: Sorted Key-Value Layout

Wide Column 的磁碟檔案通常是 SSTable 這類不可變檔案。資料會先寫入 MemTable，之後批次 flush 成按照 Key 排序的 SSTable：

```text
"Key_A:Age"  -> 25
"Key_A:Name" -> "Alice"
"Key_B:Name" -> "Bob"
```

重點是：

- 欄位可以稀疏，沒有值的欄位不必寫入。
- SSTable 寫入後不可變，更新通常是寫入新版本，而不是原地覆蓋舊資料。
- 刪除通常寫入 Tombstone，等 Compaction 時再清理。
- 寫入路徑偏向 append-only，可以把大量隨機寫入轉成順序寫入。

---

## 3. 為什麼寫入快

Wide Column 系統常搭配 LSM Tree。寫入流程通常是：

1. 寫入 Commit Log / WAL，確保持久性。
2. 寫入 MemTable，這是記憶體中的有序結構。
3. MemTable 滿了以後，flush 成不可變的 SSTable。
4. 背景 Compaction 將多個 SSTable 合併，清理舊版本與 Tombstone。

```text
Write
  -> Commit Log
  -> MemTable
  -> flush to SSTable
  -> background Compaction
```

這和 B+Tree 的核心差異在於：

| 面向 | Wide Column / LSM Tree | RDBMS / B+Tree |
| --- | --- | --- |
| 寫入方式 | Append-only，先寫 Log 與 MemTable | 找到對應 Page 後修改 |
| 磁碟行為 | 以順序寫入為主 | 容易出現隨機 I/O |
| 更新方式 | 寫入新版本，之後 Compaction 清理 | 在既有 Page 上更新 |
| Page Split | 無 B+Tree Page Split 問題 | Page 滿時可能分裂 |
| 寫入吞吐 | 通常較高 | 受索引維護與隨機寫入影響 |

所以 Wide Column 很適合大量寫入、查詢模式固定的場景，例如 chat message、IoT event、log、timeline、activity feed。

---

## 4. 查詢代價

Wide Column 的代價是查詢彈性弱。它不是為任意條件查詢設計的，而是要求先根據查詢模式設計 Key。

常見限制：

- 不擅長 Join。
- 不擅長 ad-hoc query。
- 查詢通常必須帶 Partition Key。
- Secondary Index 很容易變成 scatter-gather。
- 讀取可能需要合併 MemTable 與多個 SSTable，產生 Read Amplification。
- Tombstone 過多會拖慢讀取與 Compaction。

因此它的設計原則是：

```text
先決定查詢，再設計表。
```

這和 RDBMS 的正規化設計不同。RDBMS 通常先建出一致的資料模型，再透過 SQL、Index、Join 滿足多種查詢；Wide Column 則常為不同查詢建立不同表，接受反正規化與資料重複。

---

## 5. 適合場景

### Chat Message

查詢模式通常是：

```text
查某個 channel / conversation 的最新 N 則訊息
```

適合的 Key：

```sql
PRIMARY KEY ((conversation_id), message_id)
```

`conversation_id` 決定 Partition，`message_id` 使用 Snowflake / ULID 這類大致按時間排序的 ID，讓同一個對話內可以依時間範圍掃描。

### Time-Series / IoT

查詢模式通常是：

```text
查某個 device 在某天或某段時間的資料
```

適合的 Key：

```sql
PRIMARY KEY ((device_id, date), timestamp)
```

`date` 放進 Partition Key 可以避免單一 device 的 Partition 無限變大。

### Activity Feed / Timeline

查詢模式通常是：

```text
查某個 user 的最新動態
```

適合的 Key：

```sql
PRIMARY KEY ((user_id), created_at)
```

如果單一 user 可能非常熱門，就需要進一步做 bucket，例如加入 `bucket_date` 或 shard suffix，避免 hot partition。

---

## 6. 常見陷阱

| 問題 | 原因 | 後果 |
| --- | --- | --- |
| Hot Partition | Partition Key 太集中 | 單節點或單 Partition 過載 |
| Wide Partition | 單一 Partition 太大 | 查詢、Compaction、Repair 變慢 |
| Tombstone 過多 | 大量刪除或 TTL 到期 | 讀取掃到大量無效資料 |
| Secondary Index 濫用 | 沒有 Partition Key 的查詢 | Scatter-gather，效能不穩 |
| 查詢模式改變 | 表是 query-driven | 需要新增表或重寫資料 |

---

## 7. 一句話總結

Wide Column 的本質是把資料建模成可排序、可分片的多維 Key-Value。它用 LSM Tree 與 append-only 寫入換取極高寫入吞吐，但代價是查詢彈性低，必須先根據查詢模式設計 Partition Key 與 Clustering Key。

---

## 8. 延伸比較

- [Wide Column vs Elasticsearch Doc Values](../wide-column-vs-doc-values/README.md)：Wide Column 是以 Row / Partition 為中心的寬列資料模型；Doc Values 是 Elasticsearch / Lucene 為排序、聚合與 script 存取設計的磁碟列式結構。
