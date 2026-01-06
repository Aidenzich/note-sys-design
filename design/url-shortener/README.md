# URL Shortener 系統設計

URL Shortener 是系統設計面試的經典題目，涵蓋了編碼、資料庫設計、快取、高可用等核心概念。

## 1. 需求分析

### 1.1 功能需求

| 功能 | 說明 |
|:---|:---|
| **縮短 URL** | 輸入長 URL，返回短 URL |
| **重定向** | 訪問短 URL 時重定向到原始 URL |
| **自訂短碼** | (可選) 用戶指定短碼 |
| **過期時間** | (可選) 設定 URL 過期時間 |
| **統計分析** | (可選) 點擊次數、地區分佈 |

### 1.2 非功能需求

| 需求 | 目標 |
|:---|:---|
| **寫入量** | 100 million URLs/day ~ 1,160 writes/sec |
| **讀取量** | 10:1 讀寫比 ~ 11,600 reads/sec |
| **延遲** | < 100ms |
| **可用性** | 99.9% |
| **URL 長度** | 儘可能短 |

### 1.3 容量估算

```
每日新增: 100M URLs
保留時間: 5 年
總 URL 數: 100M × 365 × 5 = 182.5 Billion

每條記錄大小 ≈ 500 bytes (原始 URL + 短碼 + Metadata)
總儲存: 182.5B × 500B ≈ 91 TB
```

---

## 2. 短碼生成策略

### 2.1 編碼選擇

| 編碼 | 字符集 | 長度 N 可表示數量 |
|:---|:---|:---|
| Base62 | `[0-9a-zA-Z]` | $62^N$ |
| Base64 | `[0-9a-zA-Z+/]` | $64^N$ (URL 不友善) |
| Base58 | 無 `0OIl` | $58^N$ (避免混淆) |

**Base62 長度計算：**
```
62^6 = ~56.8 Billion
62^7 = ~3.5 Trillion

7 個字符足以表示數百億個短網址
```

### 2.2 方案一：Hash + Collision 處理

```
原始 URL → MD5/SHA256 → 取前 7 字符 → Base62 編碼
         → 檢查唯一性 → 衝突則取 8-14 字符
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 同一 URL 產生相同短碼 | ❌ 衝突需重試 |
| ✅ 分散式友善 | ❌ 無法預測衝突率 |

### 2.3 方案二：自增 ID + Base62

```
MySQL Auto Increment / Snowflake ID → Base62 編碼

ID: 12345678 → Base62: "dnh6"
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 無衝突 | ❌ 可預測 (安全風險) |
| ✅ 高效 | ❌ 單點瓶頸 (需分散式 ID 生成) |

### 2.4 方案三：預生成 Key 池 (KGS - Key Generation Service)

```
┌─────────────────────────────────────────────┐
│       Key Generation Service (KGS)          │
│  ┌─────────────────────────────────────┐    │
│  │  Used Keys Table  │  Free Keys Table │    │
│  │  [abc123, ...]    │  [xyz789, ...]   │    │
│  └─────────────────────────────────────┘    │
└───────────────────┬─────────────────────────┘
                    │ 取 Key
                    ▼
              URL Service
```

**流程：**
1. KGS 預先生成大量（如 10M）短碼存入 DB
2. URL Service 從 Free 表取 Key，移至 Used 表
3. 原子操作保證唯一性

| 優點 | 缺點 |
|:---|:---|
| ✅ 零衝突 | ❌ KGS 需高可用 |
| ✅ 生成與使用解耦 | ❌ 需週期性補充 |

---

## 3. 資料庫設計

### 3.1 Schema

```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    original_url VARCHAR(2048) NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0,
    
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id)
);
```

### 3.2 資料庫選型

| 選項 | 理由 |
|:---|:---|
| **MySQL/PostgreSQL** | 讀多寫少、需要索引短碼 |
| **Cassandra** | 超大規模、寫入優先 |
| **DynamoDB** | Managed、自動擴展 |

**分片策略：**
- **Partition Key**: `short_code` (Hash Sharding)
- 保證同一短碼總是落在同一分片

---

## 4. API 設計

### 4.1 縮短 URL

```http
POST /api/v1/urls
Content-Type: application/json
Authorization: Bearer <token>

{
  "original_url": "https://example.com/very/long/url",
  "custom_alias": "mylink",  // optional
  "expires_at": "2025-12-31T23:59:59Z"  // optional
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "short_code": "abc123",
  "expires_at": "2025-12-31T23:59:59Z"
}
```

### 4.2 重定向

```http
GET /abc123
→ 301/302 Redirect to original URL
```

**301 vs 302：**
| 狀態碼 | 說明 | 影響 |
|:---|:---|:---|
| **301** | Permanent Redirect | 瀏覽器會快取，減少 Server 負載 |
| **302** | Temporary Redirect | 每次都打 Server，利於統計 |

> **建議：** 需要統計時用 302，純 Redirect 用 301

---

## 5. 系統架構

```
                           ┌─────────────┐
                           │    CDN      │
                           └──────┬──────┘
                                  │
                           ┌──────▼──────┐
                           │ Load Balancer│
                           └──────┬──────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
    ┌─────▼─────┐          ┌──────▼─────┐          ┌─────▼─────┐
    │ URL Svc 1 │          │ URL Svc 2  │          │ URL Svc 3 │
    └─────┬─────┘          └──────┬─────┘          └─────┬─────┘
          │                       │                       │
          └───────────────────────┼───────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
              ┌─────▼─────┐               ┌─────▼─────┐
              │   Redis   │               │  Database │
              │  (Cache)  │               │ (MySQL/   │
              │           │               │ Cassandra)│
              └───────────┘               └───────────┘
```

### 5.1 讀取流程

```
1. GET /abc123
2. Check Redis Cache
   ├─ Hit → Redirect (301/302)
   └─ Miss → Query DB
          ├─ Found → Update Cache → Redirect
          └─ Not Found → 404
```

### 5.2 快取策略

```
Short Code → Original URL

Cache Pattern: Cache-Aside
TTL: 24 小時
Eviction: LRU
```

**熱門 URL 優化：**
- 使用 Local Cache (如 Caffeine) + Redis
- 熱門 URL 直接在 CDN 層 Redirect

---

## 6. 擴展性考量

### 6.1 分散式 ID 生成

| 方案 | 說明 |
|:---|:---|
| **Snowflake** | Twitter 方案，64-bit 包含時間戳 |
| **UUID v7** | 時間排序的 UUID |
| **DB Sequences** | 每個分片獨立 ID 範圍 |

### 6.2 讀取優化

| 技術 | 效果 |
|:---|:---|
| **Redis Cache** | 減少 DB 查詢 |
| **Read Replicas** | 分散讀取壓力 |
| **CDN Cache** | 301 Redirect 可被 CDN 快取 |

### 6.3 地理分佈

```
Global DNS → GeoDNS
  ├→ US users → US Data Center
  ├→ EU users → EU Data Center
  └→ Asia users → Asia Data Center
```

---

## 7. 安全性

| 威脅 | 對策 |
|:---|:---|
| **短碼可預測** | 使用隨機 Base62，非連續數字 |
| **惡意 URL** | 整合 Google Safe Browsing API |
| **Rate Limiting** | 限制單一 IP 的創建頻率 |
| **隱私** | 點擊統計不洩露用戶資訊 |

---

## 8. Reference
- [System Design Interview - Alex Xu, Chapter 8](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)
- [Designing a URL Shortening Service - Grokking](https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers)
