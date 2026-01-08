# Distributed Transactions (分散式事務)

在微服務架構中，一個業務操作可能跨越多個服務和資料庫，傳統的 ACID 事務不再適用。本章節涵蓋分散式事務的主要解決模式。

## 1. 問題本質

**場景：** 電商下單流程

```
Order Service ──→ 創建訂單
Payment Service ──→ 扣款
Inventory Service ──→ 減庫存

問題：如何保證這三個操作要麼全部成功，要麼全部回滾？
```

---

## 2. Two-Phase Commit (2PC)

### 2.1 流程

```
                      Coordinator
                          │
            Phase 1: Prepare (投票)
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
       Participant A  Participant B  Participant C
        (準備好了)      (準備好了)      (準備好了)
           │              │              │
           └──────────────┼──────────────┘
                          ▼
            Phase 2: Commit (提交)
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
       Participant A  Participant B  Participant C
         (提交!)         (提交!)         (提交!)
```

### 2.2 詳細步驟

| Phase | Coordinator | Participant |
|:---|:---|:---|
| **Prepare** | 發送 `PREPARE` 給所有參與者 | 執行本地事務，寫入 Redo/Undo Log，鎖定資源，回覆 `READY` 或 `ABORT` |
| **Commit** | 若全部 `READY`，發送 `COMMIT` | 正式提交，釋放鎖 |
| **Abort** | 若任一 `ABORT`，發送 `ROLLBACK` | 回滾本地事務 |

### 2.3 問題

| 問題 | 說明 |
|:---|:---|
| **同步阻塞** | 資源被鎖定直到事務完成 |
| **單點故障** | Coordinator 掛掉，參與者永遠等待 |
| **數據不一致** | Commit 階段網路分區可能導致部分提交 |
| **效能差** | 多次網路往返，鎖定時間長 |

> **實務應用：** 僅在 **強一致性必須** 且 **參與者數量少** 的場景使用，如跨庫轉帳。

---

## 3. Saga Pattern

### 3.1 概念

- **核心思想：** 將長事務拆分為一系列本地事務，每個本地事務都有對應的「補償操作」
- 本地是指：**單一微服務內部的資料庫事務**。
    - **獨立提交**：T1 做完就直接 `Commit` 到自己的資料庫，不等待其他服務。
    - **不跨服務**：它只保證自己家門內的 ACID，不與 T2、T3 共享同一個資料庫連接或鎖定。
- 一旦進入補償，T4 就會被「直接跳過」且不可使用。
    - **流程轉向**：系統由「前進模式」切換為「撤銷模式」。
    - **目標變更**：此時的目標已從「完成任務」變成「恢復原狀」，因此後續的正向動作（T4）絕對不會執行。


```
T1 → T2 → T3 → T4 (成功路徑)
         ↓
        C3 → C2 → C1 (補償路徑，當 T3 失敗時)
```

### 3.2 兩種編排模式

#### 3.2.1 Choreography (事件驅動)

```
┌────────────┐    OrderCreated    ┌────────────┐
│   Order    │ ──────Event──────→ │  Payment   │
│  Service   │                    │  Service   │
└────────────┘                    └──────┬─────┘
                                         │ PaymentCompleted
                                         ▼
┌────────────┐   InventoryReserved ┌────────────┐
│  Shipping  │ ←─────Event─────── │ Inventory  │
│  Service   │                    │  Service   │
└────────────┘                    └────────────┘
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 服務間低耦合 | ❌ 難以追蹤全局狀態 |
| ✅ 無中心點故障 | ❌ 複雜流程難以理解 |

#### 3.2.2 Orchestration (集中編排)

```
                    ┌──────────────┐
                    │  Saga        │
                    │ Orchestrator │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
     ┌──────────┐   ┌──────────┐   ┌──────────┐
     │  Order   │   │ Payment  │   │Inventory │
     │ Service  │   │ Service  │   │ Service  │
     └──────────┘   └──────────┘   └──────────┘
```

| 優點 | 缺點 |
|:---|:---|
| ✅ 流程清晰集中 | ❌ Orchestrator 是關鍵服務 |
| ✅ 易於監控和調試 | ❌ 增加耦合 |

### 3.3 範例：Order Saga

```
Forward Flow:
1. Order Service: createOrder() → 訂單狀態 = PENDING
2. Payment Service: processPayment()
3. Inventory Service: reserveStock()
4. Order Service: confirmOrder() → 訂單狀態 = CONFIRMED

Compensating Flow (當 Step 3 失敗):
1. Payment Service: refundPayment()
2. Order Service: cancelOrder() → 訂單狀態 = CANCELLED
```

### 3.4 補償操作設計

| 原則 | 說明 |
|:---|:---|
| **冪等性** | 補償操作可能被執行多次，必須冪等 |
| **順序相反** | 按相反順序執行補償 |
| **可重試** | 補償失敗時需能重試 |
| **無法補償的操作** | 如發送 Email、簡訊，需特殊處理 (不補償或人工介入) |

---

## 4. TCC (Try-Confirm-Cancel)

### 4.1 三階段

| 階段 | 說明 |
|:---|:---|
| **Try** | 預留資源 (如: 凍結餘額、鎖定庫存) |
| **Confirm** | 確認操作 (如: 實際扣款、實際減庫存) |
| **Cancel** | 取消預留 (如: 解凍餘額、釋放庫存) |

### 4.2 流程示意

```
Try Phase:
┌──────────────────────────────────────────────────────┐
│ Payment: freezeBalance(100)  ← 凍結 100 元           │
│ Inventory: reserveStock(1)   ← 預留 1 件商品         │
└──────────────────────────────────────────────────────┘
         │ 全部 Try 成功
         ▼
Confirm Phase:
┌──────────────────────────────────────────────────────┐
│ Payment: deductBalance(100)  ← 實際扣款              │
│ Inventory: reduceStock(1)    ← 實際減庫存            │
└──────────────────────────────────────────────────────┘

如果 Try 失敗:
Cancel Phase:
┌──────────────────────────────────────────────────────┐
│ Payment: unfreezeBalance(100)                        │
│ Inventory: releaseStock(1)                           │
└──────────────────────────────────────────────────────┘
```

### 4.3 TCC vs Saga

| 特性 | TCC | Saga |
|:---|:---|:---|
| **資源鎖定** | Try 階段鎖定 | 無鎖定，立即生效 |
| **隔離性** | 較好 (資源被預留) | 較差 (中間狀態可見) |
| **業務侵入** | 高 (需實現 3 個 API) | 中 (需實現補償 API) |
| **適用場景** | 短事務、強隔離 | 長事務、最終一致 |

---

## 5. Outbox Pattern (發件箱模式)

### 5.1 問題

**場景：** 需要同時更新資料庫並發送事件

```python
def create_order(order):
    db.save(order)           # 1. 寫入 DB
    kafka.send(order_event)  # 2. 發送事件
    # 問題: 如果 Step 1 成功但 Step 2 失敗？
```

### 5.2 解法：Outbox Table

```
┌─────────────────────────────────────┐
│          Database Transaction       │
│  ┌─────────────┐  ┌──────────────┐  │
│  │ Orders      │  │   Outbox     │  │
│  │ (業務表)     │  │  (待發事件)   │  │
│  └─────────────┘  └──────────────┘  │
└─────────────────────────────────────┘
                          │
                          ▼ 異步讀取
              ┌────────────────────────┐
              │  Outbox Poller/CDC     │
              │  (Debezium, Polling)   │
              └───────────┬────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │   Message Broker       │
              │   (Kafka, RabbitMQ)    │
              └────────────────────────┘
```

### 5.3 實作

```sql
-- 1. 同一個交易內寫入
BEGIN;
INSERT INTO orders (id, user_id, amount) VALUES (1, 101, 100);
INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload)
VALUES (uuid(), 'Order', 1, 'OrderCreated', '{"id":1,"amount":100}');
COMMIT;

-- 2. Poller 定期讀取 Outbox 並發送事件
SELECT * FROM outbox WHERE sent = false ORDER BY created_at LIMIT 100;

-- 3. 發送成功後標記
UPDATE outbox SET sent = true WHERE id = ?;
```

### 5.4 優點

| 優點 | 說明 |
|:---|:---|
| ✅ 強一致性 | DB 和事件在同一交易 |
| ✅ 可重試 | Outbox 保證不丟失 |
| ✅ At-Least-Once | 事件至少發送一次 |

---

## 6. 模式選擇指南

| 場景 | 推薦模式 |
|:---|:---|
| 跨服務操作，可容忍最終一致 | **Saga (Choreography)** |
| 複雜業務流程，需要可視化監控 | **Saga (Orchestration)** |
| 金融轉帳，需要強隔離 | **TCC** |
| 同庫跨表，需要原子性 | **2PC** (資料庫層) |
| 需要可靠發送事件 | **Outbox Pattern** |

---

## 7. Reference
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [Debezium - Outbox Pattern](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [ByteByteGo - Saga vs 2PC](https://blog.bytebytego.com/p/saga-vs-2pc)
