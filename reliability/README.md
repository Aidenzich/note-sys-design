# Reliability Patterns (可靠性模式)

在分散式系統中，網路不穩定和服務故障是常態。本章節涵蓋保護系統彈性的核心模式。

## 1. Circuit Breaker (熔斷器)

### 1.1 概念

**問題：** 當下游服務故障時，持續發送請求會：
- 累積大量超時，佔用連線池
- 拖慢整個系統
- 阻止下游恢復 (被請求淹沒)

**解法：** 像電路保險絲一樣，當故障達到閾值時「斷開」

### 1.2 狀態機

```
                    ┌──────────────────────────┐
                    │         CLOSED           │ ← 正常運作
                    │  (允許請求通過)           │
                    └────────────┬─────────────┘
                                 │ 連續失敗 N 次
                                 ▼
                    ┌──────────────────────────┐
                    │          OPEN            │ ← 熔斷中
                    │  (直接拒絕，不發請求)     │
                    └────────────┬─────────────┘
                                 │ 等待 Timeout
                                 ▼
                    ┌──────────────────────────┐
                    │       HALF-OPEN          │ ← 試探期
                    │  (允許少量請求測試)       │
                    └─────────┬────────────────┘
                        成功 │        │ 失敗
                             ▼        ▼
                          CLOSED     OPEN
```

### 1.3 關鍵參數

| 參數 | 說明 | 典型值 |
|:---|:---|:---|
| `failure_threshold` | 觸發熔斷的連續失敗次數 | 5-10 |
| `success_threshold` | Half-Open 時恢復所需成功次數 | 3-5 |
| `timeout` | Open 狀態持續時間 | 30s-60s |
| `half_open_max_calls` | Half-Open 時允許的試探請求數 | 3-10 |

### 1.4 實作範例 (Python Tenacity 風格)

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Circuit is open")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

### 1.5 Fallback 策略

當熔斷觸發時的後備方案：

| 策略 | 說明 | 適用場景 |
|:---|:---|:---|
| **返回快取** | 返回最後成功的結果 | 資料不敏感的場景 |
| **返回預設值** | 返回空列表、0 等 | 非必要功能 |
| **降級服務** | 呼叫備用服務 | 有備援系統 |
| **排隊重試** | 將請求放入佇列稍後處理 | 非即時需求 |
| **直接拒絕** | 返回錯誤給用戶 | 必須即時的功能 |

---

## 2. Retry with Backoff (退避重試)

### 2.1 重試策略

| 策略 | 公式 | 說明 |
|:---|:---|:---|
| **Fixed Delay** | `delay = 1s` | 固定間隔 |
| **Linear Backoff** | `delay = attempt * 1s` | 線性增長 |
| **Exponential Backoff** | `delay = 2^attempt * base` | 指數增長 |
| **Exponential + Jitter** | `delay = random(0, 2^attempt * base)` | **推薦** |

### 2.2 為什麼需要 Jitter?

**問題：** 當服務恢復時，所有 Client 同時重試會再次打爆服務。

```
Without Jitter:
Time 0: Service recovers
Time 1s: 1000 clients retry simultaneously → Service crashes again

With Jitter:
Time 0: Service recovers
Time 0.1s~2s: Clients retry scattered → Service handles gracefully
```

### 2.3 實作範例

```python
def retry_with_exponential_backoff(func, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff with full jitter
            delay = random.uniform(0, (2 ** attempt) * base_delay)
            delay = min(delay, 60)  # Cap at 60 seconds
            time.sleep(delay)
```

### 2.4 何時該重試 / 不該重試

| 應該重試 | 不應重試 |
|:---|:---|
| 網路超時 | 400 Bad Request (Client 錯誤) |
| 503 Service Unavailable | 401/403 認證/授權失敗 |
| 429 Too Many Requests | 409 Conflict (資源衝突) |
| Connection Reset | 422 業務邏輯錯誤 |
| DNS Resolution Failure | 500 且確定是 Bug |

---

## 3. Timeout (超時控制)

### 3.1 超時類型

| 類型 | 說明 | 典型值 |
|:---|:---|:---|
| **Connection Timeout** | 建立 TCP 連線的時限 | 1-5s |
| **Read Timeout** | 等待回應的時限 | 5-30s |
| **Write Timeout** | 發送請求的時限 | 5-10s |
| **Request Timeout** | 整個請求的總時限 | 10-60s |
| **Idle Timeout** | 連線閒置的時限 | 30-120s |

### 3.2 級聯超時

**問題：** 鏈式呼叫時，如何設定超時？

```
Client → A (timeout=10s) → B (timeout=?) → C (timeout=?)
```

**原則：** 上游超時 > 下游超時總和

```
A: timeout = 10s
  └→ B: timeout = 4s
       └→ C: timeout = 2s

A 的 10s 需 > B(4s) + buffer(2s)
B 的 4s 需 > C(2s) + buffer(1s)
```

### 3.3 Deadline Propagation

更好的做法是傳遞 **Deadline** 而非 Timeout：

```
Client sets: deadline = now + 10s

A receives: deadline = now + 10s
A calls B with: deadline (still the same absolute time)

B receives: deadline = now + 8s (2s already elapsed)
B calls C with: deadline

C receives: deadline = now + 6s
```

**實作：** gRPC 原生支援 Deadline Propagation

---

## 4. Bulkhead (艙壁隔離)

### 4.1 概念

**靈感來源：** 船艙隔板防止一處進水導致全船沉沒

**IT 場景：** 隔離不同的資源池，避免一個功能故障拖垮整個系統

### 4.2 實作方式

**方式一：Thread Pool Isolation**

```
┌─────────────────────────────────────────────┐
│                 Application                 │
│  ┌──────────────┐  ┌──────────────┐        │
│  │ Payment Pool │  │ Catalog Pool │        │
│  │ (10 threads) │  │ (20 threads) │        │
│  └──────┬───────┘  └──────┬───────┘        │
│         │                  │                │
│         ▼                  ▼                │
│  ┌──────────────┐  ┌──────────────┐        │
│  │Payment Service│  │Catalog Service│       │
│  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────┘

Payment 服務爆滿 → 只有 Payment Pool 受影響
Catalog 服務正常運作
```

**方式二：Semaphore Isolation**

```python
# 限制同時進行的請求數
payment_semaphore = Semaphore(10)

def call_payment():
    if payment_semaphore.acquire(blocking=False):
        try:
            return payment_service.charge()
        finally:
            payment_semaphore.release()
    else:
        raise BulkheadFullError("Payment service is overloaded")
```

### 4.3 Thread Pool vs Semaphore

| 特性 | Thread Pool | Semaphore |
|:---|:---|:---|
| 隔離程度 | 完全隔離 (獨立 Thread) | 共用 Thread Pool |
| 資源開銷 | 高 (每個池獨立 Thread) | 低 |
| 適用場景 | 嚴格隔離需求 | 輕量級限制 |
| Timeout 控制 | 可以 | 困難 |

---

## 5. Rate Limiter (限流)

*(詳見 [Rate Limiting 章節](../api-design/rate-limiting/README.md))*

---

## 6. 模式組合：韌性架構

### 6.1 推薦組合

```
Request
   │
   ▼
┌─────────────────┐
│  Rate Limiter   │ ← 入口限流
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Timeout       │ ← 總時限
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Bulkhead      │ ← 資源隔離
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Circuit Breaker │ ← 熔斷保護
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Retry       │ ← 失敗重試
└────────┬────────┘
         │
         ▼
   Downstream Service
```

### 6.2 業界工具

| 語言/平台 | 工具 |
|:---|:---|
| Java | Resilience4j, Hystrix (維護模式) |
| Go | go-kit, Sentinel-Go |
| Python | Tenacity, PyBreaker |
| .NET | Polly |
| Service Mesh | Istio, Envoy (自動處理) |

---

## 7. Reference
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [AWS - Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Resilience4j Documentation](https://resilience4j.readme.io/)
