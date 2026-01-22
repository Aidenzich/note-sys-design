# Amazon SQS (Simple Queue Service)

Amazon SQS 是一種完全受管 (Fully Managed) 的訊息佇列服務，用於解耦 (Decouple) 和擴展微服務、分散式系統和無伺服器應用程式。

## 核心概念

### Standard Queue (標準佇列)
- **吞吐量**：幾乎無限 (Unlimited Throughput)。
- **順序性**：不保證順序 (Best-effort ordering)，訊息可能會亂序到達。
- **重複訊息**：至少一次傳遞 (At-least-once delivery)，極少數情況下可能會收到重複訊息。

### FIFO Queue (先進先出佇列)
- **吞吐量**：有限制 (預設每秒 300 條訊息，可透過 batching 提升至 3000)。
- **順序性**：嚴格保證先進先出 (First-In-First-Out)。
- **重複訊息**：剛好一次處理 (Exactly-once processing)。

---

## 主要限制 (Limitations)

1.  **訊息大小 (Message Size)**
    - 單條訊息上限為 **256 KB**。
    - 若需傳輸更大型的資料，通常搭配 S3 使用 (將資料存入 S3，僅在 SQS 傳遞 S3 path - 此模式稱為 Claim Check Pattern)。

2.  **保留期限 (Retention)**
    - 訊息最長保留 **14 天**。
    - 預設保留 4 天。
    - 一旦被 Consumer 處理並刪除後，無法再次讀取 (無 Replay 能力)。

3.  **延遲 (Latency)**
    - 雖然很快，但通常不如某些專用的低延遲系統 (如 ZeroMQ)。
    - 依賴 Polling (輪詢) 機制：
        - **Short Polling**：立即返回 (可能空手而回)。
        - **Long Polling** (推薦)：等待直到有訊息或超時 (減少空請求，降低成本)。

4.  **In-flight Messages**
    - "In-flight" 指的是已被 Consumer 接收但尚未被刪除或超時的訊息。
    - Standard Queue: 上限 120,000。
    - FIFO Queue: 上限 20,000。

---

## SQS vs Kafka 主要區別

Kafka 與 SQS 雖然都是訊息傳遞系統，但設計理念完全不同：**SQS 是 Queue (佇列)**，而 **Kafka 是 Log (日誌)**。

### 1. Fan-out (扇出) 能力
這是 SQS 最弱的環節之一。
- **Kafka**：天生支援 Fan-out。多個 Consumer Groups 可以訂閱同一個 Topic，各自讀取同一份資料，互不干擾。
- **SQS**：是 **Point-to-Point** 模型。一條訊息被一個 Consumer 接收後，就會變得不可見 (Visibility Timeout)，處理完後刪除。其他 Consumer 無法再讀到該訊息。
    - **解決方案**：若要在 AWS 上實現 Fan-out，通常需要 **SNS + SQS** 組合 (Fanout Pattern)。即先將訊息發送到 SNS Topic，由 SNS 複製並推送到多個訂閱該 Topic 的 SQS Queues。

### 2. 資料持久性與 Replay (重播)
- **Kafka**：持久化存儲訊息一段時間 (Retention Policy)，即使訊息被消費過，只要還在保留期內，都可以重設 Offset 進行 **Replay (重跑資料)**。
- **SQS**：訊息一旦被 Consumer 確認完成 (DeleteMessage)，就**永久消失**。無法回溯或重跑。

### 3. 消費模式 (Push vs Pull)
- **Kafka**：Consumer **主動 Pull** 資料 (雖然各個 SDK 封裝得像是在不斷接收)。
- **SQS**：Consumer **主動 Pull** (ReceiveMessage API)。
    - *註：在 AWS Lambda 觸發器模式下，AWS 內部元件會幫你 Poll SQS 並 Invoke Lambda，這讓 SQS 看起來像 Push，但本質仍是 polling。*

### 4. 順序保證
- **Kafka**：在同一個 **Partition** 內保證順序。
- **SQS**：
    - Standard: 不保證順序。
    - FIFO: 全局有序 (或依據 Message Group ID 有序)，但性能受限。

### 總結比較表

| 特性 | Amazon SQS | Apache Kafka |
| :--- | :--- | :--- |
| **模型** | Queue (佇列) | Log (日誌流) |
| **主要用途** | 任務派發、解耦、緩衝 | 即時串流處理、Event Sourcing、Log Aggregation |
| **Fan-out** | **弱** (需搭配 SNS) | **強** (Consumer Groups) |
| **訊息狀態** | 處理後刪除 (Destructive) | 可配置保留期 (Persistent/Replayable) |
| **吞吐量** | 高 (Standard)、中 (FIFO) | 極高 |
| **維運難度** | 無 (Serverless) | 高 (需管 Broker/ZK) 或用 MSK |
| **最大訊息** | 256 KB | 預設 1MB (可調整，但建議也不要太大) |
