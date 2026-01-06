# API Design 原則與最佳實踐

本章節涵蓋 RESTful API 設計原則、版本控制策略、冪等性設計，以及 GraphQL 與 gRPC 的對比。

## 1. RESTful API 成熟度模型 (Richardson Maturity Model)

| Level | 名稱 | 特徵 | 範例 |
|:---|:---|:---|:---|
| **Level 0** | The Swamp of POX | 單一 URL，用 body 區分操作 | `POST /api` + `{"action": "getUser"}` |
| **Level 1** | Resources | 使用不同 URL 代表資源 | `/users`, `/orders` |
| **Level 2** | HTTP Verbs | 正確使用 HTTP 動詞 | `GET /users/1`, `DELETE /users/1` |
| **Level 3** | HATEOAS | 回應中包含相關操作連結 | `{"links": [{"rel": "self", "href": "/users/1"}]}` |

> **業界實務：** 大多數 API 達到 Level 2 即可，Level 3 (HATEOAS) 較少被採用。

---

## 2. URL 設計規範

### 2.1 基本原則

| 原則 | 正確 ✅ | 錯誤 ❌ |
|:---|:---|:---|
| 使用名詞 (資源) | `/users`, `/orders` | `/getUsers`, `/createOrder` |
| 使用複數 | `/users/123` | `/user/123` |
| 使用小寫 | `/user-profiles` | `/UserProfiles` |
| 使用連字符 | `/user-profiles` | `/user_profiles` |
| 避免動詞 | `POST /orders` | `/orders/create` |
| 層級關係 | `/users/123/orders` | `/orders?user_id=123` (也可接受) |

### 2.2 特殊操作

當 CRUD 無法表達時，可使用 **動詞子資源**：

```
POST /orders/123/cancel    ← 取消訂單
POST /users/123/subscribe  ← 訂閱
POST /reports/generate     ← 生成報告 (非標準資源)
```

---

## 3. HTTP 方法與冪等性

### 3.1 方法對應表

| 方法 | 用途 | 冪等性 | 安全性 | Request Body | Response Body |
|:---|:---|:---|:---|:---|:---|
| `GET` | 讀取資源 | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| `POST` | 新增資源 | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| `PUT` | 完整替換 | ✅ Yes | ❌ No | ✅ Yes | Optional |
| `PATCH` | 部分更新 | ❌ No* | ❌ No | ✅ Yes | Optional |
| `DELETE` | 刪除資源 | ✅ Yes | ❌ No | Optional | Optional |

> *PATCH 可以設計為冪等 (如使用 JSON Patch)，但標準不強制。

### 3.2 冪等性的重要性

**問題場景：** 網路超時重試，如何避免重複扣款？

```
Client → POST /payments → (超時) → 重試 → 雙倍扣款 💀
```

**解法：Idempotency Key**

```http
POST /payments
Idempotency-Key: abc123-unique-request-id
Content-Type: application/json

{"amount": 100, "currency": "USD"}
```

**Server 處理邏輯：**
```python
def process_payment(idempotency_key, payment):
    # 1. 檢查是否已處理過
    existing = redis.get(f"idempotency:{idempotency_key}")
    if existing:
        return existing  # 返回之前的結果
    
    # 2. 處理支付
    result = payment_service.charge(payment)
    
    # 3. 儲存結果 (設定過期時間)
    redis.setex(f"idempotency:{idempotency_key}", 86400, result)
    return result
```

---

## 4. 狀態碼使用

### 4.1 常用狀態碼

| 狀態碼 | 含義 | 使用場景 |
|:---|:---|:---|
| `200 OK` | 請求成功 | GET 成功、PUT/PATCH 更新成功 |
| `201 Created` | 資源已創建 | POST 創建成功 |
| `204 No Content` | 成功但無內容 | DELETE 成功 |
| `400 Bad Request` | 請求格式錯誤 | 參數驗證失敗 |
| `401 Unauthorized` | 未認證 | 未提供 Token |
| `403 Forbidden` | 無權限 | Token 有效但權限不足 |
| `404 Not Found` | 資源不存在 | 找不到指定資源 |
| `409 Conflict` | 資源衝突 | 唯一鍵重複、版本衝突 |
| `422 Unprocessable Entity` | 語義錯誤 | 格式正確但業務邏輯錯誤 |
| `429 Too Many Requests` | 限流觸發 | Rate Limit 超過 |
| `500 Internal Server Error` | 伺服器錯誤 | 未預期的錯誤 |
| `503 Service Unavailable` | 服務不可用 | 維護中、過載 |

### 4.2 400 vs 422

| 400 Bad Request | 422 Unprocessable Entity |
|:---|:---|
| 請求格式錯誤/無法解析 | 格式正確但語義錯誤 |
| JSON 語法錯誤 | age: -5 (邏輯上不合理) |
| 缺少必要欄位 | email 格式不正確 |

---

## 5. 分頁策略

### 5.1 Offset-based (偏移分頁)

```http
GET /users?page=3&limit=20
```

**回應：**
```json
{
  "data": [...],
  "pagination": {
    "page": 3,
    "limit": 20,
    "total": 1000,
    "total_pages": 50
  }
}
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 可隨機跳頁 | ❌ 大 offset 效能差 (`OFFSET 10000`) |
| ✅ 直覺 | ❌ 併發新增/刪除時資料可能重複或遺漏 |

### 5.2 Cursor-based (游標分頁)

```http
GET /users?limit=20&cursor=eyJpZCI6MTAwfQ==
```

**回應：**
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIwfQ==",
    "has_more": true
  }
}
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 效能穩定 (無 offset) | ❌ 無法跳頁 |
| ✅ 適合即時資料流 | ❌ 實作較複雜 |

**Cursor 通常是 Base64 編碼的物件：**
```json
{"id": 120, "created_at": "2024-01-15T10:00:00Z"}
```

### 5.3 選擇建議

| 場景 | 推薦 |
|:---|:---|
| 後台管理系統 | Offset (需跳頁功能) |
| 無限滾動 (Mobile Feed) | Cursor |
| 資料量 < 10,000 | 皆可 |
| 資料量 > 100,000 | Cursor |

---

## 6. 版本控制策略

| 策略 | 範例 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **URL Path** | `/v1/users` | 明確、容易 debug | 污染 URL |
| **Query Param** | `/users?version=1` | 靈活 | 容易遺漏 |
| **Header** | `Accept: application/vnd.api.v1+json` | URL 乾淨 | 不易 debug |
| **Content Negotiation** | `Accept: application/vnd.myapp+json;v=1` | RESTful | 複雜 |

**業界主流：URL Path (`/v1/`, `/v2/`)** — 簡單、易於路由、方便監控

---

## 7. 錯誤回應格式

### 7.1 標準結構

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "age",
        "message": "Must be a positive integer"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

### 7.2 設計原則

1. **統一格式**：成功與失敗回應結構一致
2. **不暴露內部資訊**：`500` 錯誤不應包含 Stack Trace
3. **提供 Request ID**：方便追蹤問題
4. **機器可讀的 Code**：`VALIDATION_ERROR` 而非 `"驗證失敗"`

---

## 8. REST vs GraphQL vs gRPC

| 特性 | REST | GraphQL | gRPC |
|:---|:---|:---|:---|
| **協議** | HTTP/1.1, HTTP/2 | HTTP | HTTP/2 |
| **資料格式** | JSON (通常) | JSON | Protocol Buffers |
| **Schema** | OpenAPI/Swagger (可選) | 強制 Schema | 強制 .proto |
| **Over-fetching** | 常見問題 | ✅ 客戶端指定欄位 | N/A |
| **Under-fetching** | 需多次請求 | ✅ 單次請求多資源 | N/A |
| **即時更新** | SSE, WebSocket | ✅ Subscriptions | ✅ Streaming |
| **效能** | 普通 | 中等 | ⚡ 極高 |
| **學習曲線** | 低 | 中 | 高 |
| **適用場景** | 通用 Web API | 前端驅動、複雜查詢 | 內部微服務通訊 |

---

## 9. 章節連結
- [Rate Limiting 設計](./rate-limiting/README.md)
- [API Gateway 架構](./gateway/README.md)
