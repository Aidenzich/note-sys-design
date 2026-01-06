# Rate Limiter 系統設計

Rate Limiter 是保護系統免受過載和濫用的關鍵組件。本章節從系統設計角度探討如何設計一個分散式 Rate Limiter。

> 演算法細節請參考 [Rate Limiting 演算法](../../api-design/rate-limiting/README.md)

## 1. 需求分析

### 1.1 功能需求

| 功能 | 說明 |
|:---|:---|
| **限制請求頻率** | 每個用戶/IP/API Key 在時間窗口內的請求上限 |
| **多維度限制** | 可按 IP、User、API、Endpoint 等維度 |
| **靈活規則** | 不同 Endpoint 不同限制 |
| **通知被限流方** | 返回明確的錯誤與重試資訊 |

### 1.2 非功能需求

| 需求 | 目標 |
|:---|:---|
| **低延遲** | < 1ms 額外延遲 |
| **高可用** | 99.99% |
| **分散式** | 跨多個 Server 一致限流 |
| **精確度** | 可接受少量超限 (Soft Limit) |

### 1.3 容量估算

```
假設：
- 1000 萬 DAU
- 平均每用戶每天 100 次請求
- QPS: 10M × 100 / 86400 ≈ 11,600 QPS

Rate Limiter 需處理: 11,600 QPS × 2 (讀+寫 Counter) ≈ 23,000 Redis Ops/sec
```

---

## 2. 架構選擇

### 2.1 位置選擇

```
方案一：API Gateway 層 (推薦)
┌──────┐    ┌──────────────┐    ┌─────────────┐
│Client│───→│ API Gateway  │───→│   Services  │
│      │←───│(Rate Limiter)│←───│             │
└──────┘    └──────────────┘    └─────────────┘

方案二：Middleware 層
┌──────┐    ┌──────────┐    ┌────────────┐    ┌─────────┐
│Client│───→│   LB     │───→│Rate Limiter│───→│ Service │
│      │←───│          │←───│ Middleware │←───│         │
└──────┘    └──────────┘    └────────────┘    └─────────┘

方案三：Service 內嵌
┌──────┐    ┌─────────────────────────────────┐
│Client│───→│ Service (with Rate Limiter Lib) │
│      │←───│                                 │
└──────┘    └─────────────────────────────────┘
```

| 位置 | 優點 | 缺點 |
|:---|:---|:---|
| API Gateway | 集中管理、一致性 | Gateway 成為瓶頸 |
| Middleware | 靈活、可獨立部署 | 增加延遲 |
| 內嵌 | 最低延遲 | 更新需重新部署服務 |

### 2.2 集中式 vs 分散式

**集中式 (推薦)：**
```
Service 1 ─┐
Service 2 ─┼──→ Redis Cluster ←── 所有 Counter 在這
Service 3 ─┘
```

**分散式 (Local + Gossip)：**
```
Service 1 [Local Counter] ←──→ Gossip Protocol ←──→ Service 2 [Local Counter]
                          ←──→                ←──→
                               Service 3 [Local Counter]
```

---

## 3. 高層架構

```
                         ┌─────────────────────────────┐
                         │      Rule Configuration     │
                         │  (DB / Config Service)      │
                         └──────────────┬──────────────┘
                                        │ Sync
                                        ▼
┌──────┐    ┌───────────────────────────────────────────────┐
│      │    │            Rate Limiter Service              │
│      │───→│  ┌─────────────┐    ┌───────────────────┐    │
│Client│    │  │ Rule Engine │───→│  Counter Manager  │    │
│      │←───│  │ (In-Memory) │    │                   │    │
│      │    │  └─────────────┘    └─────────┬─────────┘    │
└──────┘    │                               │              │
            └───────────────────────────────┼──────────────┘
                                            │
                                    ┌───────▼───────┐
                                    │ Redis Cluster │
                                    │  (Counters)   │
                                    └───────────────┘
```

---

## 4. 核心組件設計

### 4.1 規則引擎 (Rule Engine)

```yaml
# 規則配置範例
rules:
  - name: "Global API Limit"
    key: "ip:{remote_ip}"
    limit: 1000
    window: 60s  # 每分鐘 1000 請求
    
  - name: "User Limit"
    key: "user:{user_id}"
    limit: 100
    window: 60s
    
  - name: "Sensitive Endpoint"
    path: "/api/v1/payments"
    key: "user:{user_id}:payment"
    limit: 10
    window: 60s
```

**規則優先級：**
1. Path-specific rules (最高)
2. User-level rules
3. IP-level rules
4. Global default (最低)

### 4.2 Counter Manager

**使用 Sliding Window Counter 演算法：**

```lua
-- Redis Lua Script (Sliding Window Counter)
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

-- 上一個窗口的 Key
local prev_key = key .. ":prev"
local prev_window_start = math.floor((now - window) / window) * window
local curr_window_start = math.floor(now / window) * window

-- 獲取計數
local prev_count = tonumber(redis.call('GET', prev_key) or 0)
local curr_count = tonumber(redis.call('GET', key) or 0)

-- 計算加權
local prev_weight = (prev_window_start + window - (now - window)) / window
local count = prev_count * prev_weight + curr_count

if count >= limit then
    return {0, math.ceil(limit - count)}  -- Rejected, remaining
end

-- 增加當前窗口計數
redis.call('INCR', key)
redis.call('EXPIRE', key, window * 2)

return {1, math.ceil(limit - count - 1)}  -- Allowed, remaining
```

### 4.3 本地快取 (規則熱快取)

```python
class RateLimiter:
    def __init__(self):
        self.rules_cache = {}  # In-memory rule cache
        self.local_counters = {}  # Optional: local counter for performance
    
    def check(self, request):
        rule = self.get_rule(request)
        key = self.build_key(rule, request)
        
        # 可選：先檢查本地 Counter
        if self.local_counters.get(key, 0) > rule.limit * 0.9:
            # 達到 90% 本地限制，強制查 Redis
            pass
        
        allowed, remaining = self.redis_check(key, rule)
        return allowed, remaining
```

---

## 5. 高可用設計

### 5.1 Redis 故障處理

| 策略 | 行為 | 適用場景 |
|:---|:---|:---|
| **Fail Open** | Redis 不可達時放行 | 可用性優先 |
| **Fail Close** | Redis 不可達時拒絕 | 安全性優先 |
| **Fallback to Local** | 切換到本地限流 | 平衡方案 |

```python
def check_rate_limit(key, limit):
    try:
        return redis.check(key, limit)
    except RedisConnectionError:
        # Fail Open: 放行但記錄日誌
        logger.warning("Redis unavailable, fail open")
        return True
```

### 5.2 多 Redis 節點同步

**問題：** 多個 Data Center / Redis Cluster 如何保持計數一致？

**解法：** 最終一致性 + 預算分配

```
Total Limit: 1000 req/min per user

Redis (US): 500 req/min quota
Redis (EU): 300 req/min quota
Redis (Asia): 200 req/min quota

定期同步剩餘配額
```

---

## 6. 回應設計

### 6.1 HTTP Headers

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1609459230

Content-Type: application/json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 30 seconds.",
    "retry_after": 30
  }
}
```

### 6.2 成功請求也返回限流資訊

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1609459230
```

---

## 7. 監控與告警

### 7.1 關鍵指標

| 指標 | 說明 |
|:---|:---|
| `rate_limit.requests.allowed` | 通過的請求數 |
| `rate_limit.requests.rejected` | 被拒絕的請求數 |
| `rate_limit.latency` | 限流檢查延遲 |
| `rate_limit.redis.errors` | Redis 錯誤數 |
| `rate_limit.rules.count` | 規則總數 |

### 7.2 告警規則

```yaml
alerts:
  - name: High Rejection Rate
    condition: rate_limit.requests.rejected / total > 0.1
    severity: warning
    
  - name: Redis Latency Spike
    condition: rate_limit.latency.p99 > 10ms
    severity: critical
    
  - name: Redis Connection Failure
    condition: rate_limit.redis.errors > 100
    severity: critical
```

---

## 8. 進階功能

### 8.1 動態限流

根據系統負載動態調整：

```python
def get_dynamic_limit(base_limit):
    cpu_usage = get_system_cpu()
    if cpu_usage > 80:
        return base_limit * 0.5  # 高負載時降低限制
    elif cpu_usage > 60:
        return base_limit * 0.75
    return base_limit
```

### 8.2 優先級佇列

VIP 用戶優先處理：

```python
if user.tier == "enterprise":
    limit = 10000
elif user.tier == "pro":
    limit = 1000
else:
    limit = 100
```

### 8.3 Graceful Degradation

接近限制時返回 Stale Data：

```python
if rate_limit.remaining < 10:
    return cache.get_stale(request)  # 返回舊資料
```

---

## 9. Reference
- [System Design Interview - Alex Xu, Chapter 4](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)
- [Stripe Rate Limiters](https://stripe.com/blog/rate-limiters)
- [Kong Rate Limiting](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
