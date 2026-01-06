# Rate Limiting (流量限制)

Rate Limiting 是保護系統免受過載攻擊、確保公平使用資源的關鍵機制。

## 1. 為什麼需要 Rate Limiting?

| 目的 | 說明 |
|:---|:---|
| **防止 DDoS** | 避免惡意請求打垮系統 |
| **成本控制** | 限制 API 調用防止帳單爆炸 |
| **公平使用** | 避免單一用戶佔用過多資源 |
| **保護下游** | 避免雪崩效應擴散到其他服務 |

---

## 2. 主流演算法比較

### 2.1 Token Bucket (令牌桶)

**原理：** 桶內以固定速率填充 Token，請求消耗 Token，桶滿時多餘 Token 丟棄。

```
             ┌─────────────────┐
             │ Token Bucket    │ ← 固定速率填充 (r tokens/sec)
             │ [●●●●●○○○○○]    │
             │  容量 = 10       │
             └────────┬────────┘
                      │ 請求拿 Token
                      ▼
              ┌───────────────┐
              │   Request     │
              └───────────────┘

Bucket 有 Token → 通過
Bucket 空 → 拒絕或等待
```

**參數：**
- `capacity (b)`: 桶容量 (決定突發上限)
- `refill_rate (r)`: 每秒填充 Token 數 (決定平均速率)

**特點：**
| 優點 | 缺點 |
|:---|:---|
| ✅ 允許突發流量 (burst) | ❌ 需維護 Token 狀態 |
| ✅ 平滑流量 | ❌ 需要精確計算 |

**適用場景：** API Gateway (AWS API Gateway, Kong)

---

### 2.2 Leaky Bucket (漏桶)

**原理：** 請求進入桶中排隊，以固定速率流出處理。

```
             Requests
                │
                ▼
         ┌────────────┐
         │  Queue     │ ← 溢出時丟棄
         │[■■■■■■■■]  │
         └─────┬──────┘
               │ 固定速率流出
               ▼
         ┌────────────┐
         │  Process   │
         └────────────┘
```

**特點：**
| 優點 | 缺點 |
|:---|:---|
| ✅ 輸出速率完全穩定 | ❌ 不允許突發 |
| ✅ 防止下游過載 | ❌ 突發時延遲增加 |

**適用場景：** 需要完全平滑輸出的場景 (如消息佇列消費者)

---

### 2.3 Fixed Window Counter (固定窗口計數)

**原理：** 將時間劃分為固定窗口 (如每分鐘)，並計數。

```
        Window 1 (00:00-01:00)    Window 2 (01:00-02:00)
        ┌────────────────────┐   ┌────────────────────┐
        │ Count: 98/100 ✅   │   │ Count: 0/100       │
        └────────────────────┘   └────────────────────┘
```

**實現 (Redis)：**
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return 0  -- 拒絕
else
    return 1  -- 通過
end
```

**問題：** **邊界突發**

```
Window 1: 00:00-01:00    Window 2: 01:00-02:00
          ......[100次]||[100次]......
                   └─┬─┘
            在 00:59-01:01 這 2 秒內有 200 次！
```

---

### 2.4 Sliding Window Log (滑動窗口日誌)

**原理：** 記錄每次請求的 Timestamp，計算過去 N 秒內的請求數。

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
    return 1  -- 通過
else
    return 0  -- 拒絕
end
```

**特點：**
| 優點 | 缺點 |
|:---|:---|
| ✅ 精確，無邊界問題 | ❌ 記憶體消耗大 (每個請求一條記錄) |
| ✅ 平滑限流 | ❌ 高流量時 O(N) 操作 |

---

### 2.5 Sliding Window Counter (滑動窗口計數器)

**原理：** 結合 Fixed Window 和 Sliding Window，用加權計算近似滑動窗口。

```
Current time: 01:15 (窗口 = 60s)

Window 0 (00:00-01:00): count_prev = 80
Window 1 (01:00-02:00): count_curr = 20

過去 60 秒估算:
  = count_prev * (45/60) + count_curr
  = 80 * 0.75 + 20
  = 80
```

**特點：**
| 優點 | 缺點 |
|:---|:---|
| ✅ 記憶體效率高 (只存 2 個計數) | ❌ 是近似值，不完全精確 |
| ✅ 基本解決邊界問題 | ❌ 實作稍複雜 |

---

### 2.6 演算法比較總表

| 演算法 | 記憶體 | 精確度 | 突發處理 | 複雜度 | 適用場景 |
|:---|:---|:---|:---|:---|:---|
| Token Bucket | 低 | 高 | ✅ 允許 | 中 | API Gateway |
| Leaky Bucket | 低 | 高 | ❌ 禁止 | 低 | 穩定輸出 |
| Fixed Window | 極低 | 低 | ⚠️ 邊界問題 | 低 | 簡單場景 |
| Sliding Log | 高 | 極高 | ✅ 精確 | 高 | 安全審計 |
| Sliding Counter | 低 | 中 | ✅ 近似 | 中 | **推薦通用** |

---

## 3. 分散式 Rate Limiting

### 3.1 挑戰

| 挑戰 | 說明 |
|:---|:---|
| **集中 vs 分散** | 集中式 Redis 精確但增加延遲；本地計數快但不精確 |
| **時鐘同步** | 不同 Server 的時間可能不一致 |
| **網路分區** | Redis 不可達時如何處理 |

### 3.2 架構模式

**模式一：集中式 (Central Store)**

```
  API Server 1 ─┐
  API Server 2 ─┼──→ Redis ← 所有計數在這
  API Server 3 ─┘
```
- ✅ 精確
- ❌ Redis 成為瓶頸/單點

**模式二：本地 + 同步 (Local + Sync)**

```
  API Server 1 [Local Counter] ──→ 定期同步
  API Server 2 [Local Counter] ──→ 定期同步 ──→ Redis (彙總)
  API Server 3 [Local Counter] ──→ 定期同步
```
- ✅ 低延遲
- ❌ 短時間可能超限

**模式三：Sticky Session**

```
  User A ──→ 總是到 Server 1 ──→ Local Limit
  User B ──→ 總是到 Server 2 ──→ Local Limit
```
- ✅ 無需同步
- ❌ 依賴 Load Balancer 配合

---

## 4. Rate Limit Response

### 4.1 HTTP Headers 規範

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1609459200

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 60 seconds."
  }
}
```

| Header | 說明 |
|:---|:---|
| `Retry-After` | 建議重試的秒數 (標準 Header) |
| `X-RateLimit-Limit` | 窗口內允許的總請求數 |
| `X-RateLimit-Remaining` | 當前窗口剩餘請求數 |
| `X-RateLimit-Reset` | 窗口重置的 Unix Timestamp |

### 4.2 最佳實踐

1. **不要隱藏限流**：明確告知用戶限制規則
2. **提供 Retry-After**：讓 Client 正確處理重試
3. **避免 Hard Fail**：考慮 Soft Limit (警告) + Hard Limit (拒絕)

---

## 5. 多層 Rate Limiting

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Edge (CDN/WAF)                                     │
│   - IP-based: 1000 req/s per IP                             │
│   - Geo-blocking                                            │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│ Layer 2: API Gateway                                        │
│   - User/API Key: 100 req/min                               │
│   - Endpoint-specific: /search = 10 req/min                 │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│ Layer 3: Application / Service                              │
│   - Business rule: 5 orders/day per user                    │
│   - Tenant quota: 1M API calls/month                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 進階策略

### 6.1 動態限流

根據系統負載動態調整限制：

```python
def get_limit():
    cpu_usage = get_cpu_usage()
    if cpu_usage > 80:
        return 50   # 高負載時限制更嚴
    elif cpu_usage > 60:
        return 75
    else:
        return 100  # 正常限制
```

### 6.2 優先級限流

| 用戶等級 | 限制 |
|:---|:---|
| Free | 100 req/min |
| Pro | 1000 req/min |
| Enterprise | 10000 req/min |

### 6.3 Graceful Degradation

當接近限制時，降級而非直接拒絕：
1. 返回快取資料
2. 簡化回應 (移除非必要欄位)
3. 降低精度 (每分鐘更新改為每 5 分鐘)

---

## 7. Reference
- [Stripe - Rate Limiting](https://stripe.com/docs/rate-limits)
- [Kong Rate Limiting Plugin](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [Cloudflare Rate Limiting](https://developers.cloudflare.com/waf/rate-limiting-rules/)
