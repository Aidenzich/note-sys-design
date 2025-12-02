# Question & Answer

面試題目模擬：設計一個大規模即時通訊系統 (Design a Massive Scale Chat System)
面試官： 「請你設計一個類似 Discord 或 Facebook Messenger 的後端系統。我們需要支援非常龐大的用戶量，並且對即時性和資料完整性有很高的要求。」
#### 1. 功能需求 (Functional Requirements)
* **一對一聊天 (1:1 Chat)**：支援低延遲訊息傳送。
* **群組聊天 (Group Chat)**：
    * 支援小群組（< 500 人）。
    * **關鍵點：** 必須支援 **超大群組 (例如萬人以上的大頻道)**。
    
    <details>
    <summary>Tip</summary>

    這點會導向筆記中的「混合路由策略」與「讀擴散」
    </details>
* **在線狀態 (User Presence)**：需顯示好友是否在線（Online/Offline）。
    <details>
    <summary>Tip</summary>

    這點會導向筆記中的「Presence Server 解耦」
    </details>
* **多裝置同步 (Multi-device Sync)**：用戶在手機已讀，電腦端也要同步已讀。
    <details>
    <summary>Tip</summary>

    這點會導向筆記中的「Cursor-based Sync」
    </details>
* **訊息持久化 (History)**：聊天記錄需永久保存，且支援跨裝置拉取歷史訊息。

#### 2. 非功能需求 (Non-Functional Requirements)
* **極高併發 (High Concurrency)**：
    * **DAU (日活躍用戶)**：5,000 萬 (50M)。
    * **同時在線 (Concurrent Users)**：**1,000 萬 (10M)**。
    <details>
    <summary>Tip</summary>

    這個數字逼迫你放棄 Long Polling，必須選用 WebSocket
    </details>
* **寫入吞吐量 (Write Throughput)**：
    * 假設平均每人每天發 20 條訊息 -> 10 億條訊息/天 (1B msg/day)。
    * 峰值 QPS 可能達到 **200k - 500k QPS**。
    <details>
    <summary>Tip</summary>

    這個數字逼迫你放棄 RDBMS，必須選用 Cassandra/HBase 這種 LSM-tree 資料庫
    </details>
* **低延遲 (Low Latency)**：
    * 訊息傳遞延遲需 < **50ms (P99)**。
    <details>
    <summary>Tip</summary>

    這個限制逼迫你思考 Hot Path，不能經過 Kafka 這種 Disk-based queue，必須走 Redis
    </details>
* **資料一致性 (Reliability)**：
    * **Zero Message Loss**：訊息絕對不能丟失。
    <details>
    <summary>Tip</summary>

    這會導向筆記中的 Dual-Path：雖然熱路徑走 Redis，但必須有冷路徑寫入 DB
    </details>





### 題目的隱藏意圖
#### 1. 「1000 萬同時在線」 vs. WebSocket
* **面試官意圖：** 如果你用傳統 HTTP Long Polling，Server 需要維護 1000 萬個 Thread 或者頻繁握手，機器成本會爆炸。
* **筆記回應：** 所以採用 **WebSocket**（Stateful 連線），並使用 Go/Erlang/Netty 這種能處理高併發連線的技術。

#### 2. 「延遲 < 50ms」 vs. Redis Pub/Sub (Hot Path)
* **面試官意圖：** 訊息來了，如果要先寫入硬碟 (Kafka/DB) 再通知接收者，光是 I/O 就可能超過 50ms。
* **筆記回應：** 設計 **Dual-Path**。熱路徑直接走 **Redis Pub/Sub (In-memory)** 秒傳；冷路徑才走 Kafka 慢慢入庫。這就是為什麼筆記裡有「Redis vs Kafka」的激烈辯論。

#### 3. 「萬人大群組」 vs. Inbox/Group Channel 混合路由
* **面試官意圖：** 如果萬人的群組，一個人講話，你要把這則訊息 Copy 20 萬份塞進每個人的信箱 (Inbox) 嗎？那資料庫會瞬間被寫掛。
* **筆記回應：** 所以筆記提到了 **「讀擴散 (Pull)」** 機制，大群組不寫 Inbox，而是讓用戶去訂閱群組 ID。

#### 4. 「10 億條訊息/天 + 永久儲存」 vs. Cassandra/HBase
* **面試官意圖：** 一年就是 3650 億條資料。RDBMS 的 B+Tree 隨著資料量變大，插入性能會急劇下降（索引分裂）。你需要一個寫入性能是 O(1) 的資料庫。
* **筆記回應：** 選擇 **Cassandra/ScyllaDB (LSM-Tree)**，因為它是為寫入優化的，且支援無限的水平擴展。


## Follow UPs
### 第一階段：架構與選型（Testing Architectural Intuition）
**Q1. 「在訊息路由（Message Routing）的設計上，很多人會直接想到用 Kafka 來做 Message Queue，你覺得這是一個好主意嗎？為什麼？」**
* **筆記對應點：** `Topic Explosion`、`Redis Pub/Sub vs Kafka`。
* **面試官預期的答案：**
    * **絕對不行**（或至少要指出這不是 Kafka 的強項）。
    * 關鍵字：**Topic Explosion（Topic 爆炸）**。
    * 解釋：Kafka 的 Topic 是實體檔案，無法為每個 User/Group 建立一個 Topic。
    * 解釋：Kafka 是 Pull 模型，而 Chat 需要 Push 模型。
    * 正確解法：Redis Pub/Sub 做路由，Kafka 做後端非同步任務（Pipeline）。

**Q2. 「我們需要儲存海量的歷史訊息，你會選擇什麼資料庫？為什麼 Discord 後來放棄了 MongoDB 改用 Cassandra/ScyllaDB？」**
* **筆記對應點：** `Storage Design` 表格、`Wide-Column NoSQL`。
* **面試官預期的答案：**
    * 選擇 **NoSQL (Wide-Column)** 如 Cassandra 或 ScyllaDB。
    * 關鍵字：**LSM Tree**（寫入優化）、**線性擴展**。
    * 反對 RDBMS：因為 Chat 是寫多讀多，且數據量隨時間線性增長，RDBMS 的 B+Tree 索引維護成本太高且分庫分表麻煩。
    * Schema 設計：Partition Key 用 ChannelID，Clustering Key 用 MessageID。


### 第二階段：深挖痛點與優化（Testing Problem Solving）
**Q3. 「如果有一個擁有 5000 個好友的網紅上線了，系統要通知這 5000 個人『他上線了』，這會造成什麼問題？你會怎麼設計來避免影響聊天功能？」**
* **筆記對應點：** `Presence Server`、`解耦`、`Fan-out`。
* **面試官預期的答案：**
    * 問題：這是 **Fan-out（寫擴散）** 問題，會佔用大量計算資源。
    * 解法：**服務解耦**。將「狀態管理」從「核心聊天服務」中拆分出來，獨立一個 `Presence Server`。
    * 即使狀態更新延遲了幾秒（最終一致性），也不能讓它阻塞核心的訊息傳遞。

**Q4. 「如果群組很大（例如 20 萬人的大群），每個人發一則訊息，系統負載會非常大，你會怎麼處理這種大群組的訊息路由？」**
* **筆記對應點：** `Inbox (寫擴散) vs Group Channel (讀擴散)`。
* **面試官預期的答案：**
    * 區分策略：小群用 Inbox 模式（寫擴散，簡單方便）；大群用 **Group Channel 模式（讀擴散）**。
    * 解釋：在大群裡，如果用 Inbox 模式，發一條訊息要寫入 20 萬個收件箱，系統會崩潰。應該只有一個 Channel，讓 20 萬個在線用戶去訂閱這個 Channel。

**Q5. 「當伺服器重啟或網路波動後，數百萬用戶同時嘗試重連（Reconnect），這時候會發生什麼事？你要怎麼保護後端？」**
* **筆記對應點：** `Thundering Herd (重連風暴)`、`Backoff + Jitter`。
* **面試官預期的答案：**
    * 關鍵字：**Thundering Herd Problem（驚群效應/重連風暴）**。
    * Client 端機制：**Exponential Backoff** (指數退避) 加上 **Jitter** (隨機抖動)。不要讓 100 萬人都在第 1.0 秒發起請求，而是分散在 1~10 秒內。

---

### 第三階段：資料一致性與細節（Testing Consistency & Details）

**Q6. 「兩個用戶同時發送訊息，我們要怎麼保證訊息的順序是正確的？為什麼不用 MySQL 的 Auto Increment ID？」**
* **筆記對應點：** `Snowflake ID`。
* **面試官預期的答案：**
    * 分散式系統中很難用中央式的 Auto Increment（有效能瓶頸）。
    * 解法：**Snowflake ID**。
    * 優點：**K-sorted** (大致按時間排序)，這對 DB (LSM Tree) 寫入效能非常重要，且能保證全域唯一。

**Q7. 「你提到用 Redis 存已讀狀態，但如果用戶一直刷已讀，Redis 寫入量也會很大，怎麼辦？」**
* **筆記對應點：** `Write-Back (寫入合併)`。
* **面試官預期的答案：**
    * 不用每次 `read` 都寫 DB。
    * 使用 **Write-Back** 策略：在 Redis 裡只更新 `last_read_message_id`，然後由 Worker 批量（例如每幾秒或累積 100 條）非同步寫入 Cassandra/DB。

