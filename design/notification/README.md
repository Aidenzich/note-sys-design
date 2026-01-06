# Push Notification 系統設計

Push Notification 是現代應用的核心功能，涉及多平台整合、高吞吐量處理和可靠投遞。

## 1. 需求分析

### 1.1 功能需求

| 功能 | 說明 |
|:---|:---|
| **多平台支援** | iOS (APNs), Android (FCM), Web (Web Push) |
| **多類型通知** | 行銷、交易、系統告警 |
| **排程發送** | 定時通知 |
| **用戶偏好** | 允許用戶設定接收偏好 |
| **追蹤分析** | 送達率、點擊率 |

### 1.2 非功能需求

| 需求 | 目標 |
|:---|:---|
| **吞吐量** | 10 Million 通知/小時 |
| **延遲** | 即時通知 < 1s，批量可接受 < 10s |
| **可靠性** | 重要通知保證送達 |
| **可擴展** | 支援百萬級同時發送 |

---

## 2. 推播服務商對比

| 服務 | 平台 | 協議 | 特點 |
|:---|:---|:---|:---|
| **APNs** | iOS/macOS | HTTP/2 | Apple 生態，憑證管理 |
| **FCM** | Android/iOS/Web | HTTP | Google 服務，支援 Topic |
| **Web Push** | Browser | HTTP | 需 VAPID Key |
| **HMS Push** | Huawei | HTTP | 華為設備必須 |

### 2.1 FCM vs APNs 架構差異

```
FCM:
App → Your Server → FCM → Device
                 ↑
        Registration Token (每設備唯一)

APNs:
App → Your Server → APNs → Device
                 ↑
        Device Token + Certificate/Key
```

---

## 3. 高層架構

```
                                  ┌─────────────────┐
                                  │ Notification    │
          API / Admin Panel ─────→│ Trigger Service │
                                  └────────┬────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Message Queue (Kafka)                    │
│  Topic: high_priority | normal_priority | low_priority          │
└─────────────────────────────┬───────────────────────────────────┘
                              │
    ┌─────────────────────────┼─────────────────────────┐
    ▼                         ▼                         ▼
┌─────────┐             ┌─────────┐             ┌─────────┐
│ Worker 1│             │ Worker 2│             │ Worker N│
│ (APNs)  │             │ (FCM)   │             │(Web Push)│
└────┬────┘             └────┬────┘             └────┬────┘
     │                       │                       │
     ▼                       ▼                       ▼
  ┌──────┐               ┌──────┐               ┌──────┐
  │ APNs │               │ FCM  │               │ Web  │
  └──────┘               └──────┘               └──────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Analytics Store  │
                    │ (ClickHouse/ES)  │
                    └──────────────────┘
```

---

## 4. 核心組件

### 4.1 Device Registry

```sql
CREATE TABLE device_tokens (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    device_token VARCHAR(512) NOT NULL,
    platform ENUM('ios', 'android', 'web') NOT NULL,
    app_version VARCHAR(20),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    
    INDEX idx_user_id (user_id),
    UNIQUE INDEX idx_device_token (device_token)
);
```

**Token 生命週期管理：**
1. **Registration**: App 啟動時上報 Token
2. **Refresh**: Token 可能變化，需定期更新
3. **Invalidation**: 收到平台回報 Token 失效時標記

### 4.2 Notification Template Service

```json
{
  "template_id": "order_shipped",
  "title": "Your order has been shipped!",
  "body": "Order #{{order_id}} is on its way to {{address}}",
  "data": {
    "order_id": "{{order_id}}",
    "tracking_url": "{{tracking_url}}"
  },
  "platform_overrides": {
    "ios": {
      "sound": "default",
      "badge": "{{unread_count}}"
    },
    "android": {
      "channel_id": "orders",
      "click_action": "OPEN_ORDER_DETAIL"
    }
  }
}
```

### 4.3 優先級佇列

```
High Priority Queue:
  - 交易確認、安全告警、OTP
  - 獨立 Worker Pool，保證延遲

Normal Priority Queue:
  - 訂單更新、社交互動
  - 共享 Worker Pool

Low Priority Queue:
  - 行銷推廣、週報
  - Batch 處理，可延遲
```

---

## 5. 可靠性設計

### 5.1 投遞保證

```
┌──────────────────────────────────────────────────────────────┐
│                    Notification Lifecycle                    │
│                                                              │
│  Created → Queued → Sent → Delivered → Opened/Clicked        │
│     │                │          │                            │
│     └──→ Failed ←────┘          └──→ Expired                │
└──────────────────────────────────────────────────────────────┘
```

**狀態追蹤：**
```sql
CREATE TABLE notification_logs (
    id BIGINT PRIMARY KEY,
    notification_id BIGINT,
    device_token VARCHAR(512),
    status ENUM('queued', 'sent', 'delivered', 'failed', 'expired'),
    platform_response TEXT,
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    
    INDEX idx_notification_id (notification_id),
    INDEX idx_status (status)
);
```

### 5.2 重試策略

```python
def send_with_retry(notification, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = send_to_platform(notification)
            return result
        except RateLimitError:
            # APNs/FCM 限流，等待後重試
            wait_time = (2 ** attempt) + random.random()
            time.sleep(wait_time)
        except InvalidTokenError:
            # Token 失效，標記並停止重試
            mark_token_invalid(notification.device_token)
            return
        except TemporaryError:
            # 暫時性錯誤，重新入隊
            requeue_with_delay(notification, delay=60)
            return
```

### 5.3 處理平台限流

| 平台 | 限制 | 策略 |
|:---|:---|:---|
| **APNs** | 無硬性限制，但會限流 | 維護連線池，指數退避 |
| **FCM** | 每設備 240 msg/min，每主題 1M/min | Topic 廣播，批次發送 |

---

## 6. 批量發送優化

### 6.1 FCM Topic Messaging

```
場景: 向 100 萬用戶發送行銷通知

方式 1: 逐個發送
  - 100 萬次 API 呼叫
  - 時間: 數小時

方式 2: FCM Topic
  - 用戶訂閱 Topic: /topics/marketing_all
  - 1 次 API 呼叫廣播
  - 時間: 秒級
```

### 6.2 Batching

```python
# FCM 支援每批 500 個 Token
def send_batch(tokens, message):
    for batch in chunks(tokens, 500):
        fcm.send_multicast(batch, message)
```

### 6.3 Fan-out on Write (預計算)

```
用戶 A 發布動態:
  不是: 即時發送 10,000 個通知
  而是: 將通知寫入 10,000 個用戶的「待發送佇列」
        Worker 異步處理每個用戶的佇列
```

---

## 7. 用戶偏好系統

```sql
CREATE TABLE notification_preferences (
    user_id BIGINT PRIMARY KEY,
    marketing_enabled BOOLEAN DEFAULT TRUE,
    order_updates BOOLEAN DEFAULT TRUE,
    social_activity BOOLEAN DEFAULT TRUE,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    timezone VARCHAR(50)
);
```

**執行邏輯：**
```python
def should_send(user, notification):
    prefs = get_preferences(user.id)
    
    # 類型開關
    if notification.type == 'marketing' and not prefs.marketing_enabled:
        return False
    
    # 勿擾時段
    user_time = get_user_local_time(user.id, prefs.timezone)
    if prefs.quiet_hours_start <= user_time <= prefs.quiet_hours_end:
        return schedule_for_later(notification, prefs.quiet_hours_end)
    
    return True
```

---

## 8. 監控與分析

### 8.1 關鍵指標

| 指標 | 說明 |
|:---|:---|
| **Delivery Rate** | 成功送達 / 總發送 |
| **Open Rate** | 打開數 / 送達數 |
| **Click-Through Rate (CTR)** | 點擊數 / 打開數 |
| **Uninstall Rate** | 發送後 24 小時解除安裝率 |
| **Latency P99** | 從請求到送達的延遲 |

### 8.2 Dashboard

```
Real-time:
- 每秒發送量 (QPS)
- 佇列積壓深度
- 平台錯誤率

Daily:
- 各平台送達率
- 用戶參與度趨勢
- Token 失效率
```

---

## 9. Reference
- [Firebase Cloud Messaging Documentation](https://firebase.google.com/docs/cloud-messaging)
- [Apple Push Notification Service](https://developer.apple.com/documentation/usernotifications)
- [System Design Interview - Alex Xu, Vol 2](https://www.amazon.com/System-Design-Interview-Insiders-Guide/dp/1736049119)
