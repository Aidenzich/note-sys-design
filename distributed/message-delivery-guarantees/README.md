# Distributed Message Delivery Semantics

本筆記記錄分散式系統中，資料在傳送者 (Sender) 與接收者 (Receiver) 之間的一致性保證。

## 1. 核心語義 (Core Semantics)
- **At-Most-Once**: 丟失沒關係，但不能重複。
- **At-Least-Once**: 重複沒關係，但不能丟失。(Kafka 預設)
- **Exactly-Once**: 不丟失、不重複。 實踐上較困難，但多數現代系統都追求此目標，例如： At-Least-Once + idempotence

## 2. 實作機制 (Mechanisms)
- **Producer Side**: 冪等性 (Idempotence)、事務 (Transactions)。
- **Consumer Side**: 隔離級別 (Isolation Level)、Offset 管理。
- **Checkpointing**: Flink/Spark 的兩階段提交機制。

## 3. 業界應用 (Industry Standards)
- **Kafka**: `transactional.id` 與 `isolation.level=read_committed`。
    | 元件 | 配置參數 | 設定值 | 目的 |
    | --- | --- | --- | --- |
    | **Producer** | `enable.idempotence` | `true` | 避免單點重複發送 |
    | **Producer** | `transactional.id` | `some-unique-id` | 開啟事務 API 支援 |
    | **Producer** | `acks` | `all` | 確保資料持久化不丟失 |
    | **Consumer** | `isolation.level` | `read_committed` | 隔離未提交或已回滾的數據 |
    | **Consumer** | `enable.auto.commit` | `false` | 由應用程式手動配合事務提交 Offset |
- **AWS SQS**: FIFO Queues vs Standard Queues。
- **CDC**: 如何將 RDB 的事務封裝進 Kafka。








