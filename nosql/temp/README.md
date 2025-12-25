# NoSQL 核心架構與原理筆記

這份筆記涵蓋了 NoSQL 與 SQL 的比較、Redis 架構模式、資料分片策略（Sharding）、底層儲存引擎（LSM Tree vs B+ Tree），以及 Cassandra 與 MongoDB 的深度對比。

## 1. SQL vs. NoSQL 對比

| 特性 | 關聯式資料庫 (SQL / RDBMS) | 非關聯式資料庫 (NoSQL) |
| --- | --- | --- |
| **代表技術** | MySQL, PostgreSQL | MongoDB, Cassandra, Redis, DynamoDB |
| **資料模型** | 結構化 (Structured)，Table/Row | 多樣化 (Key-Value, Document, Graph, Wide-Column) |
| **擴展方式** | **垂直擴展 (Scale Up)**：升級單機硬體 | **水平擴展 (Scale Out)**：增加機器節點 |
| **一致性** | 強一致性 (ACID) | 最終一致性 (BASE) / CAP 定理權衡 |
| **Schema** | 固定 Schema (Schema-on-write) | 動態 Schema / Schema-free (Schema-on-read) |
| **適用場景** | 複雜關聯查詢、交易處理 | 大數據量、高併發寫入、靈活資料結構 |

---

## 2. Redis 架構模式 (Open Source vs. AWS)

Redis 根據需求有不同的部署架構，主要分為高可用（HA）與分片（Sharding）。

### A. Open Source Redis

* **Redis Sentinel (哨兵模式)**
* **核心功能**：**高可用性 (High Availability, HA)**。
* **機制**：監控 Master 狀態，當 Master 故障時，自動將 Slave 提升為 Master。
* **限制**：寫入能力受限於單機 Master，無法水平擴展寫入。


* **Redis Cluster (叢集模式)**
* **核心功能**：**分片 (Sharding)** + 高可用性。
* **機制**：將資料分散到多個 Master 節點（Slot 機制），每個 Master 可有 Slave。
* **優勢**：可水平擴展儲存與寫入吞吐量。



### B. AWS 託管服務 (ElastiCache & MemoryDB)

* **Amazon ElastiCache**
* 支援 Memcached 與 Redis。
* **Global Datastore**：可跨 Region 複製，提供災難復原與低延遲讀取。


* **Amazon MemoryDB for Redis**
* 相容 Redis API，但底層使用分散式交易日誌，提供**強持久性 (Durability)**，資料不遺失。



---

## 3. 資料分片策略 (Sharding Strategies)

當單機無法負荷時，需將資料分散。主要有兩種策略：

### Hash Sharding (雜湊分片)

* **機制**：`Hash(key) % Node_Count`
* **優點**：資料分佈非常均勻，負載平衡佳。
* **缺點**：
* 不支援範圍查詢 (Range Query)。
* 擴充節點時需重新搬移大量資料（除非使用 **Consistent Hashing / 虛擬節點**）。



### Range Sharding (範圍分片)

* **機制**：依照 Key 的順序切分（如 A-M 在 Node 1, N-Z 在 Node 2）。
* **優點**：支援高效的範圍查詢 (e.g., `user_id > 1000`).
* **缺點**：
* 容易產生 **Hotspot (熱點)** 問題（例如使用 Timestamp 作為 Key，寫入會集中在最新的一個節點）。



---

## 4. 儲存引擎核心：LSM Tree vs. B+ Tree

這是資料庫底層最重要的效能權衡。

### B+ Tree (Read Optimized)

* **適用**：關聯式資料庫 (MySQL/InnoDB)。
* **特性**：
* **原地更新 (In-place update)**：修改資料時直接覆蓋原位置。
* **讀取優化**：樹的高度低，查詢穩定。
* **缺點**：隨機寫入 (Random Write) 效能較差，因為磁碟 Seek 次數多。



### LSM Tree (Log-Structured Merge Tree) (Write Optimized)

* **適用**：Cassandra, HBase, RocksDB, LevelDB。
* **核心組件與流程**：
1. **WAL (Write Ahead Log)**：先寫 Log 保證不丟失。
2. **MemTable (記憶體)**：資料寫入記憶體中的樹結構（有序）。
3. **Immutable SSTable (磁碟)**：MemTable 滿了之後，**Flush** 到硬碟成為不可變的排序檔案 (Sorted String Table)。
4. **Compaction (壓縮)**：背景執行，將多個小的 SSTable 合併成大的，並清理被刪除或過期的資料。


* **優化技術**：
* **Bloom Filter**：快速判斷一個 Key "絕對不存在" 或 "可能存在" 於 SSTable 中，減少無效的磁碟 I/O。


* **代價**：
* **寫入放大 (Write Amplification)**：Compaction 過程會多次重寫資料。
* **讀取放大 (Read Amplification)**：讀取時需檢查 MemTable 和多個 SSTable。



---

## 5. Wide Column Store (以 Cassandra 為例)

### 資料模型 (Data Model)

Cassandra 的 Key 設計決定了資料的分佈與排序：

1. **Partition Key (分區鍵)**：決定資料存在哪個節點 (Node)。
2. **Clustering Key (排序鍵)**：決定在該分區內，資料如何排序 (Sorted map)。

### 架構特點

* **Peer-to-Peer (無中心化)**：所有節點地位平等，無 Master/Slave 之分。
* **Gossip Protocol**：節點間透過 Gossip 協議交換狀態。
* **Secondary Index (次級索引)**：
* Cassandra 的次級索引通常是 **Local** 的（索引資料與原資料在同一節點），避免跨節點查詢。
* 查詢時若不帶 Partition Key，會導致 "Scatter-Gather"（全叢集掃描），效能極差。



---

## 6. 資料一致性與修復：Merkle Tree

* **問題**：在分散式系統中，如何快速比對不同副本 (Replica) 之間的資料是否一致？
* **解決方案**：**Merkle Tree (雜湊樹)**。
* **運作原理**：
* 將資料區塊雜湊，層層向上雜湊直到 Root Hash。
* **Anti-Entropy (反熵)**：節點間只需比對 Root Hash。
* 若 Root 不同，則向下比對子節點，快速定位差異數據，僅傳輸差異部分進行修復。
* *圖示中顯示了 8 個葉子節點組成的樹狀結構，這是 Dynamo 風格資料庫修復資料的關鍵。*



---

## 7. Cassandra vs. MongoDB 深度比較

| 比較項目 | MongoDB | Cassandra |
| --- | --- | --- |
| **架構** | **Master-Slave (Replica Set)**<br>

<br>寫入只能在 Primary，讀取可擴展。 | **Peer-to-Peer (Leaderless)**<br>

<br>多點寫入，高可用性更強，無單點故障。 |
| **儲存引擎** | WiredTiger (B+ Tree 變體) | LSM Tree (寫入優化) |
| **資料結構** | **Document (JSON/BSON)**<br>

<br>適合複雜嵌套 (Nested) 資料。 | **Wide Column**<br>

<br>適合寬表，嵌套資料需轉為 JSON 字串或 UDT，查詢受限。 |
| **索引** | 豐富的索引支援 (Geospatial, Text, etc.) | 索引功能較弱，強調主鍵查詢。 |
| **適用場景** | 快速開發、內容管理、資料結構多變。 | 極高寫入量、時序資料 (Time-series)、物聯網 (IoT)。 |

