# Uber 的遷移：從 PostgreSQL 到 MySQL

Uber 在 2016 年發表那篇著名的遷移文章時，核心論點之一正是 **PostgreSQL 的 Replication 機制（基於物理層面的 WAL）導致了嚴重的「寫入放大」（Write Amplification）問題**，這在 Uber 這種「高頻更新」的業務場景下，造成了難以承受的頻寬與效能瓶頸。

### 1. 核心差異：物理複製 vs. 邏輯複製

要理解 Uber 的痛點，首先要區分 Postgres 和 MySQL 在複製（Replication）上的根本差異：

* **PostgreSQL (當時的版本): 物理複製 (Physical Replication)**
    * Postgres 的 WAL (Write-Ahead Log) 記錄的是 **「磁碟頁面（Page）上的物理變更」**。
    * 也就是說，它記錄的是「檔案系統層級」的改動，例如：「在 Block A 的 Offset B 處寫入了這些 byte」。
    * Slave (Replica) 收到這些指令後，是對著自己的硬碟做一模一樣的物理操作。

* **MySQL: 邏輯複製 (Logical Replication)**
    * MySQL 的 Binlog (Row-based) 記錄的是 **「資料層級的變更」**。
    * 它記錄的是邏輯指令，例如：「將 Table X 中 ID 為 1 的那一行，Status 欄位改為 'Completed'」。
    * Slave 收到後，是自己去執行這個更新操作。

### 2. Uber 的場景：高頻更新 (High-Frequency Updates)

Uber 的業務特性是：**一筆訂單（Trip）的狀態會不斷改變**。
例如：`requesting` -> `finding_driver` -> `driver_assigned` -> `arrived` -> `in_progress` -> `completed`。
這意味著同一行資料（Row）會被頻繁地 Update。

### 3. 為什麼「物理 WAL」在這種場景下會爆炸？

這牽涉到 Postgres 的 **MVCC (多版本並發控制)** 實作方式：

1.  **新版本生成：** 當你在 Postgres 更新一行資料（Update）時，它並不是原地修改，而是 **標記舊的那行為「死」，並在硬碟上寫入一行全新的資料（Tuple）**。
2.  **索引的連帶效應 (The Index Problem)：**
    * 因為資料換了物理位置（寫到了新的地方），**所有指向這行資料的 Index（索引）** 全部都需要更新，讓它們指向新的位置。
    * 如果你的 Table 有 5 個索引，這 5 個索引全部都要改寫。
3.  **災難發生 (WAL 膨脹)：**
    * 因為 Postgres 的 Replication 是「物理」的，它必須把「那一行資料的新位置」加上「那 5 個索引的物理變更」全部寫入 WAL 並傳送給 Replica。
    * **結果：** 你可能只是邏輯上改了 4 bytes 的狀態欄位，但實際產生的 WAL 日誌可能高達數 KB 甚至更多（因為包含了一堆索引的物理頁面變更）。

這就導致了驚人的 **「寫入放大」（Write Amplification）**。對於 Uber 這種規模來說，Master 和 Replica 之間的網路頻寬瞬間被這些龐大的 WAL 資料塞爆，導致複製延遲（Replication Lag）嚴重，甚至無法同步。

### 4. 為什麼 MySQL 沒有這個問題？

相比之下，MySQL 的 InnoDB 引擎雖然也是 MVCC，但它的 Replication 是傳送 Binlog（邏輯層）：

* 當 Update 發生時，Master 告訴 Replica：「把 ID 1 的狀態改成 Completed」。
* **Replica 收到後，自己在本地執行這個操作。**
* Replica 自己去處理它本地的索引更新，不需要 Master 透過網路把「索引的物理變更」傳過來。
* **結果：** 網路傳輸的資料量非常小，頻寬極省。

### 5. Postgres 的反擊與現狀 (補充背景)
為了公平起見，必須提到 Postgres 其實有一個機制叫做 **HOT (Heap Only Tuples)** 來緩解這個問題。如果更新不涉及索引欄位，且該 Page 還有空間，Postgres 可以不更新索引。

但在 Uber 當時的案例中，他們指出 **HOT 機制經常失效**，原因可能是：
1.  他們更新的欄位本身就有索引。
2.  頁面已滿，必須寫到新頁面，導致 HOT 無法運作。

### 總結
Uber 放棄 Postgres 轉投 MySQL，並非 Postgres 不好，而是因為：
**Postgres 的「物理複製」機制，加上 Uber 的「高頻更新」與「多索引」資料結構，導致了複製流（Replication Stream）過大，塞爆了網路頻寬。**

這是一個經典的架構取捨案例：MySQL 的邏輯複製雖然在資料一致性保護上稍微弱一點（早年），但在大規模寫入的場景下，它對頻寬的友善程度遠勝當時的 Postgres。


## Reference
* [Uber's Migration from PostgreSQL to MySQL](https://www.uber.com/en-TW/blog/postgres-to-mysql-migration/)