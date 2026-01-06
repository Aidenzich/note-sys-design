# Cache 策略總覽 (Caching Strategies)

Cache 是系統設計中最重要的效能優化手段之一。本章節涵蓋快取策略、常見問題與解決方案。

## 1. 核心快取模式 (Core Caching Patterns)

### 1.1 Cache-Aside (Lazy Loading)
**最常見的模式**，應用程式自行管理 Cache 的讀寫。

```
讀取流程:
┌──────┐      ┌───────┐      ┌────┐
│Client│─1.Get→│ Cache │      │ DB │
│      │←2.Hit─│       │      │    │
└──────┘       └───────┘      └────┘

┌──────┐      ┌───────┐      ┌────┐
│Client│─1.Get→│ Cache │      │ DB │
│      │←2.Miss│       │      │    │
│      │────────3.Query────────→│    │
│      │←───────4.Return────────│    │
│      │─5.Set→│       │      │    │
└──────┘       └───────┘      └────┘
```

| 優點 | 缺點 |
|:---|:---|
| ✅ Cache 只存放「真正被請求」的資料 | ❌ Cache Miss 時延遲較高 (多一次 I/O) |
| ✅ 實作簡單、控制力高 | ❌ 需自行處理資料一致性 |
| ✅ Cache 故障時系統仍可運作 | ❌ 冷啟動問題 (Cold Start) |

**適用場景：** 讀多寫少、可容忍短暫不一致。

---

### 1.2 Read-Through
由 **Cache 層自動**處理 DB 讀取，對應用程式透明。

```
┌──────┐      ┌───────────────┐      ┌────┐
│Client│─Get─→│ Cache Provider│─Fetch→│ DB │
│      │←─────│  (Auto Load)  │←──────│    │
└──────┘      └───────────────┘      └────┘
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 應用程式邏輯簡化 | ❌ Cache Provider 需支援此模式 |
| ✅ 資料載入邏輯集中管理 | ❌ 首次請求仍有 Miss Penalty |

**代表實作：** Caffeine (Java), Guava LoadingCache

---

### 1.3 Write-Through
寫入時**同步**更新 Cache 與 DB，保證強一致性。

```
┌──────┐      ┌───────┐      ┌────┐
│Client│─Write→│ Cache │─Sync─→│ DB │
│      │←──Ack─│       │←─Ack──│    │
└──────┘       └───────┘      └────┘
```

| 優點 | 缺點 |
|:---|:---|
| ✅ Cache 永遠與 DB 同步 | ❌ **寫入延遲高** (需等 DB 確認) |
| ✅ 讀取時不需查 DB | ❌ 會快取「從未被讀取」的資料 (浪費空間) |

**適用場景：** 強一致性需求、讀遠多於寫。

---

### 1.4 Write-Behind (Write-Back)
寫入時**只更新 Cache**，背景異步寫入 DB。

```
┌──────┐      ┌───────┐         ┌────┐
│Client│─Write→│ Cache │         │ DB │
│      │←─Ack──│       │         │    │
└──────┘       │  ...  │─Async──→│    │
               │ (Batch)│        │    │
               └───────┘         └────┘
```

| 優點 | 缺點 |
|:---|:---|
| ✅ **極低寫入延遲** | ❌ **資料可能遺失** (Cache 故障時) |
| ✅ 可批次合併寫入，減少 DB 壓力 | ❌ 實作複雜度高 |

**適用場景：** 高頻寫入 (計數器、已讀狀態)，可容忍短暫資料丟失。

---

### 1.5 模式對比總表

| 模式 | 讀延遲 | 寫延遲 | 一致性 | 複雜度 | 典型場景 |
|:---|:---|:---|:---|:---|:---|
| **Cache-Aside** | Miss 時高 | 需手動 Invalidate | 最終 | 低 | 通用、電商商品頁 |
| **Read-Through** | Miss 時高 | N/A | 最終 | 中 | ORM 整合、Session |
| **Write-Through** | 低 | **高** | **強** | 中 | 金融交易、票務庫存 |
| **Write-Behind** | 低 | **極低** | 弱 | 高 | 計數器、日誌、已讀 |

---

## 2. 快取常見問題與解法 (Cache Pitfalls)

### 2.1 Cache Penetration (快取穿透)
**問題：** 請求大量**不存在的 Key**，每次都打到 DB。

```
攻擊者: GET user:999999999 (不存在)
        GET user:888888888 (不存在)
        ...
→ 全部 Miss，DB 被打爆
```

**解法：**
| 方案 | 原理 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **Bloom Filter** | 用位元陣列快速判斷 Key 是否「可能存在」 | 記憶體極省 (~1% 誤判) | 需預先載入 Keys |
| **Null Object Cache** | 將空結果也快取 (設短 TTL) | 簡單直接 | 佔用 Cache 空間 |

---

### 2.2 Cache Breakdown (快取擊穿)
**問題：** **熱點 Key** 過期瞬間，大量請求同時打到 DB。

```
熱點 Key: product:iphone15 (過期)
→ 10,000 個請求同時查 DB → DB 崩潰
```

**解法：**
| 方案 | 原理 | 適用場景 |
|:---|:---|:---|
| **Mutex Lock** | 第一個 Miss 的請求加鎖，其他等待 | 精確控制，適合更新頻率低 |
| **Logical Expiration** | Cache 永不過期，背景線程更新 | 高可用，適合讀遠多於寫 |
| **熱點預熱** | 啟動時預載入熱點資料 | 已知熱點 Key 的場景 |

**Mutex Lock 虛擬碼：**
```python
def get_with_lock(key):
    value = cache.get(key)
    if value is not None:
        return value
    
    # 嘗試獲取鎖
    if cache.setnx(f"lock:{key}", 1, ex=5):
        try:
            value = db.query(key)
            cache.set(key, value, ex=3600)
            return value
        finally:
            cache.delete(f"lock:{key}")
    else:
        # 等待並重試
        time.sleep(0.1)
        return get_with_lock(key)
```

---

### 2.3 Cache Avalanche (快取雪崩)
**問題：** 大量 Key **同時過期**，或 Cache Server 整體故障。

**解法：**
| 方案 | 原理 |
|:---|:---|
| **TTL 加隨機值** | `TTL = baseTime + random(0, 300)` 打散過期時間 |
| **多級 Cache** | L1 (Local) + L2 (Redis) + L3 (DB) |
| **降級策略** | Cache 全掛時返回預設值或限流 |
| **高可用部署** | Redis Sentinel / Cluster |

---

## 3. 快取一致性策略

### 3.1 先更新 DB 還是先更新 Cache？

| 策略 | 流程 | 風險 |
|:---|:---|:---|
| ❌ 先更 Cache 再更 DB | Cache ✓ → DB ✗ | **髒讀**：DB 失敗但 Cache 已更新 |
| ✅ 先更 DB 再刪 Cache | DB ✓ → Delete Cache | **短暫不一致**：刪除失敗時有舊資料 |
| ✅ 先更 DB 再更 Cache | DB ✓ → Set Cache | **併發覆蓋**：A 先寫 B 後寫，但 B 的 Cache 先到 |

**業界共識：先更新 DB，再刪除 Cache (Cache Invalidation)**

### 3.2 延遲雙刪 (Delayed Double Delete)
解決「先更 DB 再刪 Cache」的併發問題。

```
1. 刪除 Cache
2. 更新 DB
3. 延遲 N 秒 (通常 500ms ~ 1s)
4. 再次刪除 Cache ← 清除可能的髒讀
```

---

## 4. 快取驅逐策略 (Eviction Policies)

| 策略 | 原理 | 適用場景 |
|:---|:---|:---|
| **LRU** (Least Recently Used) | 移除最久未使用的 | 通用、大多數場景 |
| **LFU** (Least Frequently Used) | 移除使用次數最少的 | 熱點資料明顯的場景 |
| **TTL** (Time To Live) | 固定時間過期 | 有時效性的資料 |
| **Random** | 隨機移除 | 資料訪問均勻時 |

> Redis 4.0+ 支援 `volatile-lfu` 和 `allkeys-lfu` 策略

---

## 5. 章節連結
- [Redis 深度解析](./redis/README.md)
