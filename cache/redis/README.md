# Redis 深度解析

Redis (Remote Dictionary Server) 是最廣泛使用的 In-Memory 資料結構儲存系統，常被用作 Cache、Message Broker 和 Session Store。

## 1. 核心資料結構 (Data Structures)

### 1.1 基礎類型

| 類型 | 底層實現 | 時間複雜度 | 典型用途 |
|:---|:---|:---|:---|
| **String** | SDS (Simple Dynamic String) | O(1) GET/SET | Session、計數器、分散式鎖 |
| **List** | QuickList (ZipList + LinkedList) | O(1) Push/Pop, O(N) Index | 訊息佇列、最新動態 |
| **Hash** | HashTable / ZipList | O(1) HGET/HSET | 物件屬性存取 |
| **Set** | IntSet / HashTable | O(1) Add/Remove/Check | 標籤、唯一值集合 |
| **Sorted Set (ZSet)** | Skip List + HashTable | O(log N) Add/Remove | 排行榜、時間線 |

### 1.2 進階類型

| 類型 | 用途 | 核心操作 |
|:---|:---|:---|
| **HyperLogLog** | 基數估算 (UV 統計) | `PFADD`, `PFCOUNT` (~0.81% 誤差，12KB 固定記憶體) |
| **Bitmap** | 位元運算 (在線狀態、簽到) | `SETBIT`, `GETBIT`, `BITCOUNT` |
| **Geo** | 地理位置 (附近的人) | `GEOADD`, `GEORADIUS` (底層是 ZSet) |
| **Stream** | 訊息串流 (持久化 Pub/Sub) | `XADD`, `XREAD`, `XREADGROUP` |

---

## 2. 記憶體管理

### 2.1 記憶體優化技巧

| 技巧 | 說明 | 效果 |
|:---|:---|:---|
| **ZipList** | 小型 Hash/List/ZSet 使用緊湊編碼 | 減少 50%+ 記憶體 |
| **IntSet** | 純整數 Set 使用緊湊陣列 | 減少 70%+ 記憶體 |
| **共用物件池** | 0-9999 整數預先建立 | 減少重複物件 |
| **Key 命名規範** | `u:1001:s` 取代 `user:1001:session` | 累積節省可觀 |

### 2.2 Max Memory Policy (淘汰策略)

當記憶體達到 `maxmemory` 限制時：

| 策略 | 行為 | 適用場景 |
|:---|:---|:---|
| `noeviction` | 拒絕寫入 (預設) | 不允許資料丟失 |
| `allkeys-lru` | 所有 Key 中 LRU 淘汰 | 通用 Cache |
| `volatile-lru` | 僅淘汰設有 TTL 的 Key (LRU) | 保護永久性資料 |
| `allkeys-lfu` | 所有 Key 中 LFU 淘汰 (Redis 4.0+) | 熱點明顯的場景 |
| `volatile-ttl` | 優先淘汰 TTL 短的 | 時效性資料 |
| `allkeys-random` | 隨機淘汰 | 存取均勻時 |

---

## 3. 持久化機制 (Persistence)

### 3.1 RDB (Snapshot)

**原理：** 定期將記憶體快照儲存為二進位檔案。

```
觸發條件 (redis.conf):
save 900 1      # 900秒內至少1次寫入
save 300 10     # 300秒內至少10次寫入
save 60 10000   # 60秒內至少10000次寫入
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 緊湊、恢復快 | ❌ 可能丟失最後一次快照後的資料 |
| ✅ 適合災難恢復備份 | ❌ Fork 大量資料時可能卡頓 |

### 3.2 AOF (Append Only File)

**原理：** 記錄每一個寫入操作到日誌檔案。

```
appendfsync 策略:
- always    → 每次寫入都 fsync (最安全，最慢)
- everysec  → 每秒 fsync (推薦，最多丟 1 秒資料)
- no        → 依賴 OS flush (最快，風險最高)
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 資料遺失少 (最多 1 秒) | ❌ 檔案較大，恢復較慢 |
| ✅ 多種 fsync 策略 | ❌ 需定期 Rewrite 壓縮 |

### 3.3 混合模式 (Redis 4.0+)

```
aof-use-rdb-preamble yes
```

AOF 檔案前半段為 RDB 格式 (快速載入)，後半段為 AOF 增量 (資料完整)。  
**生產環境推薦此配置**。

---

## 4. 高可用架構

### 4.1 主從複製 (Replication)

```
                    ┌─────────┐
                    │ Master  │ (讀寫)
                    └────┬────┘
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │Replica1│ │Replica2│ │Replica3│ (唯讀)
         └────────┘ └────────┘ └────────┘
```

**同步機制：**
1. **全量同步 (Full Sync)：** Replica 首次連接，Master 發送 RDB + Backlog
2. **增量同步 (Partial Sync)：** 短暫斷線後，Master 只發送 Backlog 中的差異

### 4.2 Sentinel (哨兵)

**功能：** 監控、通知、自動故障轉移。

```
           Sentinel Quorum (至少 3 個)
         ┌────────────────────────────┐
         │  S1      S2      S3        │
         └────┬─────┬──────┬──────────┘
              │     │      │
              ▼     ▼      ▼
            Master ←→ Replica (監控 & 選舉)
```

**關鍵配置：**
```
sentinel monitor mymaster 127.0.0.1 6379 2  # 2 = quorum 數量
sentinel down-after-milliseconds mymaster 30000
sentinel failover-timeout mymaster 180000
```

### 4.3 Cluster (叢集)

**架構：** 資料分散在 **16384 個 Hash Slots** 中。

```
┌─────────────────────────────────────────────────┐
│               16384 Hash Slots                  │
│ [0-5460]      [5461-10922]     [10923-16383]   │
│    ↓               ↓                ↓          │
│ ┌──────┐       ┌──────┐         ┌──────┐       │
│ │Node A│       │Node B│         │Node C│       │
│ │Master│       │Master│         │Master│       │
│ └──┬───┘       └──┬───┘         └──┬───┘       │
│    │              │                │           │
│ ┌──▼───┐       ┌──▼───┐         ┌──▼───┐       │
│ │Replica│      │Replica│        │Replica│      │
│ └───────┘      └───────┘        └───────┘      │
└─────────────────────────────────────────────────┘
```

**Slot 分配原理：**
```
slot = CRC16(key) % 16384
```

**跨 Slot 限制：**
- 預設不支援跨 Slot 的多 Key 操作 (如 MGET key1 key2)
- 解法：使用 **Hash Tag** 強制同 Slot → `{user:1001}:name`, `{user:1001}:age`

---

## 5. 常見應用模式

### 5.1 分散式鎖 (Distributed Lock)

**基本實現：**
```lua
-- 加鎖 (Lua Script 保證原子性)
SET lock:order123 "uuid_value" NX EX 10
-- NX: 僅當不存在時設置
-- EX: 10秒過期

-- 解鎖 (必須驗證 owner)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**進階方案 - Redlock (多 Master)：**
1. 向 N 個獨立 Master 取得鎖
2. 若在有效時間內取得 > N/2 個鎖，視為成功
3. 否則釋放所有已取得的鎖

> ⚠️ **注意：** Redlock 仍存在爭議 (Martin Kleppmann 的批評)，對於強一致性需求建議使用 ZooKeeper 或 etcd。

### 5.2 Rate Limiting (限流)

**Sliding Window 實現 (Lua)：**
```lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

-- 移除窗口外的記錄
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- 計算窗口內請求數
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now .. '_' .. math.random())
    redis.call('EXPIRE', key, window)
    return 1  -- 允許
else
    return 0  -- 拒絕
end
```

### 5.3 排行榜 (Leaderboard)

```bash
# 新增/更新分數
ZADD leaderboard 1000 "player:001"
ZADD leaderboard 2500 "player:002"

# 取前 10 名 (由高到低)
ZREVRANGE leaderboard 0 9 WITHSCORES

# 查詢排名
ZREVRANK leaderboard "player:001"  # 返回 0-indexed 排名
```

---

## 6. Pub/Sub vs. Streams

| 特性 | Pub/Sub | Streams |
|:---|:---|:---|
| **持久化** | ❌ 無，離線即失 | ✅ 有，可回放歷史 |
| **Consumer Group** | ❌ 無 | ✅ 有 (類似 Kafka) |
| **ACK 機制** | ❌ 無 | ✅ 有 `XACK` |
| **延遲** | ⚡ 極低 (μs) | ⚡ 低 (ms) |
| **適用場景** | 即時通知、Chat 路由 | 任務佇列、Event Sourcing |

**Streams 範例：**
```bash
# 生產者
XADD mystream * event "user_signup" user_id 1001

# 消費者群組
XGROUP CREATE mystream mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 BLOCK 5000 STREAMS mystream >

# 確認處理完成
XACK mystream mygroup 1609459200000-0
```

---

## 7. 效能調優 Checklist

| 面向 | 建議 |
|:---|:---|
| **連線池** | 使用連線池，避免頻繁建立 TCP 連線 |
| **Pipeline** | 批次操作使用 Pipeline，減少 RTT |
| **Key 設計** | 避免 Big Key (單一 Key > 10KB 值得注意) |
| **熱點分散** | 對熱點 Key 加隨機後綴 (如 `hot_key_{1-10}`) |
| **禁用危險指令** | `rename-command KEYS ""`, `rename-command FLUSHALL ""` |
| **Slow Log** | `slowlog-log-slower-than 10000` (10ms) |

---

## 8. Reference
- [Redis 官方文件](https://redis.io/docs/)
- Redis in Action - Josiah Carlson
- [Antirez 的 Redlock 設計](https://redis.io/topics/distlock)
