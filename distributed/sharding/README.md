# 分佈式數據切分策略 (Data Partitioning Strategies)
以下是常見的分佈式數據切分策略總表，這張表只簡述了各個策略的運作原理、關鍵特性和代表技術。具體的實現細節請參閱各技術的文件。


| 切分策略 (Strategy) | 運作原理 (Mechanism) | 關鍵特性 (Key Features) | 代表技術/軟體 (Applied Technologies) |
| :--- | :--- | :--- | :--- |
| **Consistent Hashing**<br>(一致性哈希) | **環狀空間 + 虛擬節點**<br>將 Key 和 Node 都映射到一個哈希環上，數據存於順時針方向的第一個節點。 | 👍 **擴容平滑**：新增/刪除節點只影響鄰近的一小部分數據。<br>⚠️ **複雜度**：需維護虛擬節點 (Vnodes) 以解決數據傾斜。 | • **Cassandra** (Token Ring)<br>• **DynamoDB** (AWS 底層原理)<br>•  **OpenStack Swift**<br>• **Memcached** (Client 端常見實作) |
| **Hash Slots**<br>(哈希槽 / 固定分片) | **預分桶 + 映射**<br>預先切分成固定數量的 Slot (如 16384)，再將 Slot 指派給節點。`CRC(key) % SlotCount`。 <br> 節點會根據設定分配所有的Slot | 👍 **管理清晰**：數據遷移以 Slot 為單位，解耦了 Key 與 Node。<br>⚠️ **總數固定**：Slot 總數一旦定好，後期難以更改。 | • **Redis Cluster** (16384 slots)<br>• **Codis** (1024 slots) |
| **Modulo Hashing**<br>(取模哈希) | **簡單取餘數**<br>直接用公式計算節點索引：`Hash(Key) % NodeCount`。 | 👍 **保證順序** (對特定 Key)：同一個 Key 永遠去同一個節點。<br>⚠️ **擴容代價大**：節點數 N 改變，幾乎所有數據映射都會失效 (Rebalance Storm)。 | • **Kafka** (有 Key 的訊息，保證 Partition 順序)<br>• **Nginx** (預設的 Hash Load Balancing)<br>• **傳統資料庫分表** (Sharding JDBC 等早期實作) <br>• Milvus, Qdrant, Weaviate 等向量資料庫(假設有 10 個分片，每個分片會擁有總數據的 1/10。每個分片會為自己內部的數據建立一個獨立的 HNSW 索引) |
| **Range Based Partitioning**<br>(範圍分片) | **按值排序**<br>根據 Key 的實際數值大小切分成連續區間 (如 `a-f`, `g-z`)。 | 👍 **範圍查詢強**：支援 `SCAN`, `Range Query` 高效。<br>⚠️ **寫入熱點**：若 Key 單調遞增 (如時間戳)，寫入會集中在單一節點。 | • **HBase**<br>• **BigTable**<br>•  <br>•  **CockroachDB**, **TiDB**, **Google Spanner** 等newSQL<br>• **MongoDB** (預設的 **Range Sharding**) |
| **Hashed Sharding (Range on Hash)**<br>(哈希值範圍分片) | **哈希後再分範圍**<br>先對 Key 做 Hash，再將 **Hash 值** 進行範圍切分 (Chunk)。 | 👍 **解決熱點**：將單調遞增的 Key 打散，實現寫入負載均衡。<br>⚠️ **喪失範圍查詢**：範圍查詢會變成廣播 (Scatter-Gather)。 | • **MongoDB** (選用 **Hashed Sharding** 模式時)<br>*(註：這是 MongoDB 特有的混合實作，本質是 Range 架構但內容是 Hash 值)* |
| **Directory Based**<br>(目錄/查表法) | **中心化元數據**<br>不依賴算法，由一個中心服務 (NameNode) 記錄每個數據塊的具體位置。 | 👍 **靈活**：數據物理位置可隨意移動，不受 Key 限制。<br>⚠️ **單點瓶頸**：元數據服務器容易成為效能瓶頸。 | • **HDFS** (NameNode)<br>• **Ceph** (MDS - 用於 File System)<br>*(註：BigTable/HBase 也有 Meta 表，但核心是 Range)* |

### 重點歸納
1.  **MongoDB：** 是最特別的，它**跨了兩欄**。預設是 **Range Based**，但為了效能（避免熱點）可以開啟 **Hashed Sharding**（這是一種特殊的 Range on Hash 策略，不同於 Consistent Hashing）。
- **Kafka：** 使用的是 **Modulo Hashing**（取模）。這是因為 Kafka 的 Partition 通常是固定的，且必須嚴格保證 `同 Key 順序`，這比「平滑擴容」更重要。
- **Cassandra / DynamoDB：** 是 **Consistent Hashing** 的教科書代表，專注於高可用與隨時可擴充節點。
- **Redis Cluster (16384 slots):** 為什麼是 16384？這是因為 Slot 資訊需要透過 Gossip 協議在節點間交換。如果 Slot 太多 (比如 65536)，封包標頭會太大，浪費頻寬；如果太少，叢集擴容的精細度又不夠 。
- **NewSQL Sharding 選擇**：因為 NewSQL 的「SQL」屬性決定了「範圍查詢 (Range Query)」是其根本，而 Hashed Sharding 對範圍查詢的破壞力太大，因此 NewSQL 通常選擇 **Range Based**。雖然架構上預設是 Range，但為了防止熱點，這些 NewSQL 提供了 **「手動開啟的 Hashed 行為」** 被稱作 `Opt-In Hashing`：Opt-in Hashing 的本質是透過打亂 Partition Key (Primary Key) 的值，讓 Table Data 均勻隨機分佈在所有節點上，以消除寫入熱點並最大化寫入吞吐量；但同時維護 全局二級索引 (Global Secondary Index)，該索引不跟隨 Table Data 分佈，而是依據索引鍵自身的範圍 (Range) 獨立儲存在特定的節點上，藉此保留高效的範圍查詢 (Range Query) 能力
