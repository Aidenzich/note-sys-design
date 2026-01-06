# Apache Cassandra 深度解析

Cassandra 是一個無主節點 (Masterless)、對等架構 (P2P)、高可用且可線性擴展的寬列儲存 (Wide-column Store) 資料庫，最初由 Facebook 開發。

## 1. 核心架構

### 1.1 Gossip Protocol (八卦協定)

Cassandra 沒有 Master 節點，所有節點都是平等的。
- 節點間透過 Gossip 協定每秒交換狀態資訊
- 確保所有節點都知曉叢集中其他節點的存活狀態

### 1.2 Consistent Hashing (一致性雜湊)

使用 Token Ring 將資料分佈到各個節點。
- 每個節點負責 Ring 上的一段 Token Range
- **Virtual Nodes (Vnodes)**：每個實體節點擁有多個虛擬節點 (通常 256 個)，均勻分佈負載

### 1.3 複製機制 (SimpleStrategy vs NetworkTopologyStrategy)

- **SimpleStrategy**：順時針找下 N-1 個節點作為副本 (適合單機房)
- **NetworkTopologyStrategy**：意識到機架 (Rack) 和資料中心 (DC)，確保副本跨 Rack/DC 分佈

---

## 2. 寫入路徑 (Write Path) - LSM Tree

Cassandra 針對寫入進行了極致優化，寫入是順序 I/O。

1. **Commit Log**: 寫入磁碟上的 Append-only Log (Durability)
2. **MemTable**: 寫入記憶體中的 Sorted Map
3. **SSTable (Sorted String Table)**: 當 MemTable 滿時，Flush 到磁碟成為不可變的 SSTable
4. **Compaction**: 背景合併多個 SSTable，移除被標記為刪除 (Tombstone) 的資料

```
Write Request ──→ Commit Log (Disk)
              └──→ MemTable (RAM) ──Flush──→ SSTable (Disk)
```

---

## 3. 讀取路徑 (Read Path)

讀取需合併 MemTable 和多個 SSTable 的結果。

1. **Check MemTable**
2. **Check Row Cache** (Optional)
3. **Bloom Filter**: 快速判斷 Key 是否**不**在某個 SSTable 中
4. **Partition Key Cache**: 查找 Partition 在 SSTable 中的 Offset
5. **Scan SSTables**: 讀取並合併

**Read Repair**:
讀取時若發現副本間數據不一致 (基於 Checksum)，會自動修復舊數據 (後台異步或阻塞同步，視 Consistency Level 而定)。

---

## 4. 可調一致性 (Tunable Consistency)

Cassandra 允許每個讀寫請求指定一致性層級 (CL)。

### 4.1 常見層級

| Level | 寫入行為 | 讀取行為 | 強度 |
|:---|:---|:---|:---|
| **ONE** | 1 個 Replica Ack 即成功 | 返回最快那 1 個 Replica | 弱 (高可用) |
| **QUORUM** | `(N/2 + 1)` 個 Ack | 讀取 `(N/2 + 1)` 個並比較 | 強一致 (若 W+R > N) |
| **ALL** | 所有 Replica Ack | 讀取所有 Replica | 最強 (低可用) |
| **LOCAL_QUORUM** | 本地 DC 的 Quorum | 本地 DC 的 Quorum | 多 DC 優化 |

### 4.2 強一致性公式

$$W + R > N$$

- $N$: Replication Factor (副本數，如 3)
- $W$: Write Consistency Nodes
- $R$: Read Consistency Nodes

**範例 ($N=3$):**
- Write QUORUM (2) + Read QUORUM (2) = 4 > 3 ✅ 強一致
- Write ONE (1) + Read ONE (1) = 2 < 3 ❌ 最終一致

---

## 5. 資料模型設計

Cassandra 是 **Query-driven Design** (根據查詢來設計表)。

### 5.1 Primary Key 組成

`PRIMARY KEY ((Partition Key), Clustering Key)`

- **Partition Key**: 決定資料在哪個節點 (Hash)
- **Clustering Key**: 決定資料在節點內的排序 (Sorted)

### 5.2 範例：時間序列資料

需求：查詢某感測器特定時間段的數據

```sql
CREATE TABLE sensors (
    sensor_id INT,
    date DATE,
    timestamp TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((sensor_id, date), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

- Partition Key: `(sensor_id, date)` → 避免單個 sensor 數據無限增長導致 Partition 過大 (Wide Partition)
- Clustering Key: `timestamp` → 支援範圍查詢 `WHERE ... AND timestamp > ?`

---

## 6. Anti-Patterns (避坑指南)

| 模式 | 說明 | 後果 |
|:---|:---|:---|
| **Queue-like Data** | 頻繁刪除數據 | Tombstones 堆積，嚴重影響讀取效能 |
| **Wide Partition** | 單個 Partition > 100MB | 記憶體壓力大，Rebalance 慢 |
| **Secondary Index** | 濫用索引 (`CREATE INDEX`) | 分散式查詢效率極差 (需查詢所有節點) |
| **Distributed Join** | 客戶端做 Join | 效能低，應使用反正規化 (Denormalization) |

---

## 7. 維護與運維

### 7.1 Tombstones (墓碑)

Cassandra 的刪除是寫入一個標記 (Tombstone)。如果查詢範圍內有大量 Tombstones，效能會急劇下降。
- `gc_grace_seconds`: 控制 Tombstone 保留時間 (預設 10 天)

### 7.2 Nodetool

核心維護工具：
- `nodetool status`: 查看叢集狀態
- `nodetool repair`: 修復副本不一致 (Anti-entropy)
- `nodetool compact`: 手動觸發 Compaction
- `nodetool cleanup`: 節點增加後清除不屬於自己的資料

---

## 8. Reference
- [Cassandra Architecture Guide](https://cassandra.apache.org/doc/latest/architecture/)
- [ScyllaDB Architecture (Compatible but C++)](https://www.scylladb.com/product/technology/)
