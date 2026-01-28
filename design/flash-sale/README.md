# 搶票情境




## Q. 在高併發搶票場景下，為什麼你選擇 Redis Lua script 而不是 MySQL Row Lock？如果 Redis 掛了怎麼辦？

### 1. 核心決策：為什麼選擇 Redis Lua Script 放棄 MySQL Row Lock？

在 Senior 工程師的架構視角中，技術選型取決於「資源成本」與「系統隔離性」。

#### MySQL Row Lock 的致命傷 (Context Switch)
在高併發下（例如 10,000 QPS 搶同一張票），大量的 DB 連線會卡在等待鎖（Lock Wait）的狀態：
- 耗盡 DB 的 Connection Pool
- 造成 OS 層級劇烈的 **Context Switch**
- 導致資料庫 CPU 飄高，甚至引發「雪崩效應」，拖垮登入、付款等其他核心業務。

#### Redis Lua 的優勢 (Serialization & Moving Computation to Data)
* **執行位置：** Lua Script 是直接在 **Redis Server 端** 執行的（而非 App 端）。這實現了「計算向數據移動」，省去了多次網路來回（Round Trip Time，RTT）的延遲與競態條件。
* **原子性：** Redis 是單執行緒模型。透過 Lua，我們將「讀取、判斷、扣減」封裝成一個原子操作。在執行期間，**沒有其他指令可以插隊**。這讓 Redis 成為一個極高效能的「流量漏斗」，擋掉 99% 的無效請求。

> [!NOTE]
> 雖然選擇 Lua，但必須嚴格控制其時間複雜度。上線前，所有 Lua Scripts 都必須通過 Benchmark（極限壓測），確保僅包含 O(1) 或簡單的內存操作（Hash/String），並將執行時間控制在毫秒級以下，嚴格防止阻塞 Redis 主執行緒。


### 2. Failure Scenarios：如果 Redis 掛了怎麼辦？

Senior 工程師必須假設 Cache 是不可靠的。即便設定 AOF `everysec`，在 Redis 當機重啟的瞬間，仍可能發生「資料回滾」導致記憶體庫存比實際多（例如賣了 10 張，重啟後變回 5 張）。

#### 策略：Redis 負責抗壓，MySQL 負責兜底（Source of Truth）。
* **第一道防線（Redis）：** 利用 `Lua Script` 做快速篩選，只有扣減成功的請求能往下走。
* **最後防線（MySQL 的樂觀鎖變體）：**
我們不該信任 Redis 的扣減結果是最終真理。當請求流到 MySQL 寫入訂單時，我們執行「帶條件的原子更新」：

    ```sql
    -- 使用 product_id 鎖定特定商品行，並利用 stock > 0 作為最後守門員
    UPDATE inventory SET stock = stock - 1 WHERE product_id = 888 AND stock > 0;
    ```

* 如果 Redis 因為故障回滾而「超賣」放進了第 101 個人（實際庫存只有 100），當這條 SQL 在 DB 執行時，因為 `stock` 已經是 0，`stock > 0` 條件不成立。
* **結果：** 資料庫回傳 **`Rows Affected: 0`**。
* **處理：** 應用層（Application Layer）捕捉到 0 影響行數，判定為「搶購失敗」，觸發補償邏輯（回傳 User 失敗訊息），從而保證了**資料的強一致性**。

> [!NOTE]
> 為了修復 Redis 與 DB 間的狀態不一致，我們需要一個異步補償機制 (Reconciliation)。 例如：消息隊列 (MQ)：MySQL 扣減成功才發 Event，如果 Application 發現 Redis 扣了但 MySQL 沒成功，需要發送回補指令給 Redis。

定時校準 (Sync Job)：每隔幾秒鐘，檢查 Redis 庫存與 MySQL 庫存的差異，若差異過大則強制用 MySQL 覆蓋 Redis（以 DB 為準）。


#### Circuit Breaker / Local Rate Limiter
如果 Redis 掛了，我們不能讓流量直接全部透傳（Pass-through）到 DB。必須在 Application 層實施 Circuit Breaker（熔斷） 或 Local Rate Limiter（本機限流）。 例如，當 Redis 連線失敗率飆高時，應用程式直接拒絕 99% 的搶票請求，只允許極少量（如 50 QPS）進入 MySQL 進行『樂觀鎖扣減』。這是在保命（Availability）和服務（Functionality）之間的權衡。



### 3. 如何應付 HOT KEY 問題？
為了避免單個 Redis 節點過熱，我們會做 **Key Sharding（庫存分片）**，將 `product_id=888` 的 10,000 個庫存拆分成 `888_1` 到 `888_10` 分散在不同節點。

#### 尾盤問題： 為了避免單一分片售完導致的 False Negative（假性缺貨）:
* 分片 A (stock_0) 的 100 個已經被搶光了（剩 0）。
* 分片 B (stock_1) 因為運氣好，分配到的用戶較少，還剩下 10 個。
* 用戶 X 拿到的分片是 A，因此他會得到一個錯誤的「缺貨」訊息，因為 stock_1 裡面還有庫存，稱為 **False Negative**。

##### Solutions:
| 策略 | 機制 (Description) | 優點 (Pros) | 缺點 (Cons) |
| :--- | :--- | :--- | :--- |
| **策略 A：退化回單機扣減**<br>(Degrade to Single Key) | **核心概念**：最常見的做法。當庫存低於水位線（e.g. < 100）時，回收原本分散的庫存。<br><br>**機制**：<br>1. 設定水位線 (Watermark)。<br>2. 觸發 switch，將 `stock_0` ~ `stock_9` 合併回 `stock_main`。<br>3. 後續所有請求強制導向 `stock_main`。 | <span style="color: green">絕對不會有 False Negative，保證賣光。</span> | <span style="color: red">實作複雜，需要處理「回收庫存」瞬間的併發鎖問題。</span> |
| **策略 B：分片間歸集 / 輪詢**<br>(Rebalancing / Work Stealing) | **核心概念**：不合併庫存，而是跨分片「借」庫存。<br><br>**機制**：<br>1. 用戶 X 分配到 `stock_0` 發現沒貨。<br>2. 程式自動檢查 `stock_1`、`stock_2`...<br>3. 若有貨，則扣減其他分片。 | <span style="color: green">不需要複雜的「合併」動作。</span> | <span style="color: red">在高併發下，輪詢會造成 Redis 請求量瞬間暴增（放大 N 倍），可能把 Redis 打掛。</span> |



### Takeaway
這個架構的權衡（Trade-off）在於：

1. **Happy Path：** Redis 扛流量，效能極高。
2. **Disaster Path：** Redis 掛掉時，MySQL 透過 `WHERE stock > 0` 強制保證不超賣，犧牲部分使用者體驗（可能 Redis 說成功但 DB 失敗）來換取**商業數據的正確性**。

