> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

# MySQL 如何應付大量查詢流量？(Binlog, Slave DB)

隨著系統業務量增加，即便將查詢優化到極致，系統仍會負荷不了瞬間大量查詢，此時只剩垂直與水平擴充兩個選項，垂直擴充相對簡單，提升硬體 CPU 和記憶體，但缺點是升級時需要停機，因此水平擴充較為常見，<span style="color: orange">MySQL 常見的 Cluster 架構就是 Master-Slave 架構，一台 Master 處理寫入請求並同步給多台 Slave，而多台 Slave 負責查詢請求。</span>

## 為何只需要一台 Master？寫入請求不需要分流嗎？

多 Master 的 Cluster 架構較複雜且容易出錯，例如：

- 多台 Master 同時更新相同資料時，誰的結果是最新的？
- 採用 Data Sharding 會造成跨 Server Transaction 實作 ACID 困難，且：
  - 寫入比讀取花更少 CPU 和記憶體
  - 通常查詢頻率又比寫入高，需要分流的通常是查詢請求
  - 因此單 Master 多 Slave 架構能降低複雜性，並分散查詢請求提升負載上限。

### Master 如何向 Slave 同步資料？

當 Master 收到更新請求 (e.g `INSERT`, `UPDATE`, `DELETE` ) 時，需要即時將修改內容同步給 Slave，同步方式有兩種，**Push & Pull**：

#### 同步方式比較：Push vs Pull

| 同步模式 | 運作方式 | 優點 (Pros) | 缺點 (Cons) |
| :--- | :--- | :--- | :--- |
| **Master Push** | Master 主動推送更新資料給 Slave | **資料同步延遲低**<br>(Real-time) | **Master 實作複雜**<br>(需紀錄每台 Slave 進度、多 Thread 推送) |
| **Slave Pull** | Slave 主動向 Master 拉取更新資料 | **Master 邏輯單純、資源消耗低**<br>(Slave 自行管理進度，Master 僅寫入 Queue) | **資料同步延遲較高**<br>(受 Polling 間隔影響) | 

在單 Master 架構中，Master 是唯一寫入點，其穩定性與效能很重要。為了降低 Master 的複雜度與負載風險，MySQL 採用 **Slave Pull** 的設計，雖然資料同步會有些許延遲，但優點是：
- Master 專注於寫入，不需追蹤每個 Slave 的進度或管理推送邏輯。
- Slave 可根據自身能力調整拉取頻率與批次大小，在資料量大時一次拉取更多資料，反而提高處理效率。

而在拉取模式下，<span style="color:orange">Master 要將**變更資料先寫入一個暫存區**，讓不同的 Slave 依照其進度拉取資料，而這個暫存區就是 **Binlog**</span>

## 為何需要新的儲存區？不能用 Redo Log 嗎？

既然所有寫入都會先寫到 `Redo Log` 中，Slave 不能直接去 Redo Log 拉資料同步嗎？

然而，直接用 `Redo Log` 同步資料有兩個缺點：

- `Redo Log` 屬於 InnoDB 是 Storage Engine 結構，如果換一個 Engine 需要修改 Slave 同步邏輯，缺乏系統彈性。
- 資料同步至 B+Tree 後就會從 `Redo Log` 清除，不會等到所有 Slave 成功同步在清除，不符合同步所需的持久性保障。

因此 MySQL 需要在 SQL Layer 層使用 `Binlog` 結構用來當作 Slave 同步的暫存區。
![alt text](imgs/image-12.png)


#### Binlog 儲存格式比較

| 格式 (Format) | 內容 (Content) | 優點 (Pros) | 缺點 (Cons) |
| :--- | :--- | :--- | :--- |
| **STATEMENT** | **直接儲存 SQL 指令** <br> (e.g. `INSERT ...`) | **空間極小**<br>(只存指令文字) | **資料不一致風險**<br>(非確定性函數如 `NOW()`, `RAND()` 在 Slave 可能算出不同值) |
| **ROW** | **儲存修改後的資料列** <br> (Row-based changes) | **資料絕對一致**<br>(精準複製變更結果，無需重新解析 SQL) | **空間巨大**<br>(每一筆受影響的 Row 都要記錄，大量 Update 時會暴增) |
| **MIXED** | **混合模式** <br> (MySQL 自動判斷) | **平衡空間與安全**<br>(預設用 Statement，遇到非確定性函數自動切換 Row) | **不可預測性**<br>(無法完全掌握 MySQL 當下決定用哪種格式) |

## 寫入要分別同步到 Redo Log ＆ Binlog 兩個結構，會不會有同步 Redo Log 成功但 Binlog 失敗的可能？

如果 `Redo Log` 成功 `Binlog` 失敗會造成 Master 和 Slave 資料不一致，因此 MySQL 透過 **2-Phased Commit** 解決 `Redo Log` & `Binlog` 同步議題：

![alt text](imgs/image-13.png)

其核心流程如下 (確保 Redo Log 與 Binlog 一致)：

1.  **Prepare Phase (InnoDB)**:
    寫入 Redo Log 並將 Transaction 標記為 `PREPARE` 狀態。
2.  **Commit Phase (Binlog + InnoDB)**:
    *   寫入 Binlog (這是真正的 Commit 點)。
    *   再回到 InnoDB 將 Redo Log 標記為 `COMMIT`。

**Crash Recovery 邏輯**：
*   若 Crash 發生在步驟 2 之前 (Binlog 未寫入) -> **Rollback**。
*   若 Crash 發生在步驟 2 之後 (Binlog 已寫入) -> **Commit** (即便 Redo Log 還在 Prepare 狀態，只要 Binlog 有，就算成功)。

---

### Group Commit (效能優化)

原本每筆 Transaction 都要執行多次 disk fsync (Redo Log prepare + Binlog + Redo Log commit)，導致 IOPS 瓶頸。
**Group Commit** 技術則是將多個同時提交的 Transaction 聚合起來：
1.  **Queue**: 收集多個 Transaction。
2.  **Batch Write**: 一次性將它們的 Binlog 寫入磁碟 (只要一次 fsync)。
3.  **Batch Commit**: 一次性在 InnoDB 完成 Commit。

這大幅減少了 fsync 次數，顯著提升高併發下的寫入效能。

可透過下面參數控制 SQL Layer 聚合方式：
- `binlog_group_commit_sync_delay` ：等待多久 (單位為微秒) 後執行 Prepare & Fsync
- `binlog_group_commit_sync_no_delay_count` ：累積多少個 Transaction Commit 後執行 Prepare & Fsync

## Slave 如何知道要從哪裡開始同步？ (Checkpoint 機制)

為了避免故障重啟後重頭同步，Slave 必須記錄「上一次同步到哪裡」。MySQL 有兩種紀錄方式：

### 1. 傳統方式：Binlog File & Position (物理座標)

這是最早期的方式 (MySQL 5.6 以前預設)，Slave 紀錄的是「檔案名稱 + 檔案內的位移量 (Offset/Bytes)」。可以透過 `SHOW MASTER STATUS` 查看。

*   **File**: Binlog 檔名 (e.g. `mysql-bin.000003`，遞增)
*   **Position**: 寫入的 Byte Offset (下一個寫入點的起始位置)

![alt text](imgs/image-14.png)

> [!CAUTION]
> **致命缺點：Failover 困難 (座標系不一致)**
> Position 是「物理座標」，會受到 Server 配置影響 (e.g. 檔名設、Header 長度、壓縮格式)。
> 即使兩台 DB 資料完全一樣，它們記錄同一筆 Transaction 的 Position 也可能完全不同。
>  *   舊 Master: Transaction A 寫在 `pos: 500`
>  *   新 Master (Slave A): Transaction A 寫在 `pos: 502` (因為 Header 可能多 2 bytes)
>
> **災難情境**：
> 1.  舊 Master 掛掉。
> 2.  Slave B 回報它同步到 `pos: 500` (拿著舊 Master 的地圖)。
> 3.  你把 Slave B 指向新 Master (Slave A)。
> 4.  Slave B 說：「請從 `500` 之後傳給我」。
> 5.  但對新 Master 來說，`500` 可能還在 Transaction A 的資料中間 (因为它結束在 `502`)
> 6.  **結果**：MySQL 噴錯，同步中斷。需要人工介入，手動找出這筆資料在新 Master 是對應到哪行。

![alt text](imgs/image-15.png)
![alt text](imgs/image-16.png)

### 2. 現代方式：GTID (Global Transaction ID) (邏輯座標)

為了解決上述問題，MySQL 5.6 引入了 GTID。這是一個 **跨 Server 唯一的邏輯 ID**，跟檔案位置無關。

*   **格式**: `source_uuid:transaction_id`
    *   `source_uuid`: 該 Transaction 來源 Server 的唯一 ID。
    *   `transaction_id`: 遞增序號。
*   **運作**: 每筆 Transaction 都有一個專屬 ID。Slave 只要紀錄「我已經執行過哪些 ID (GTID Set)」。
*   **優點**: **自動化 Failover**。Slave 切換 Master 時，MySQL 會自動比對 GTID Set，自動補齊缺少的資料，無需人工計算 Position。

![Screenshot 2026-01-06 at 20.53.36](https://hackmd.io/_uploads/B18nHFcV-g.png)

如圖，即便 File Location & Position 不同，但 **Executed_Gtid_set** 是相同的，這讓 Cluster 管理變得簡單許多。

**GTID 相關變數**：
- `Retrieved_Gtid_Set`: 從 Master 拉下來的 GTID 集合 (Relay Log)。
- `Executed_Gtid_Set`: 已經執行並寫入 Binlog 的 GTID 集合。
- `Purged_Gtid_Set`: 已經被清除 (Purge) 掉的 GTID 集合。

## Slave 是怎麼同步資料進硬碟的？

Slave 從 Master Binlog 拉取資料後並不會直接寫入 InnoDB (Redo Log)，而是先循序寫入到 **Relay Log** 中，這主要有兩個考量：

1.  **效能與解耦 (Decoupling)**：
    *   **寫入成本低**：Relay Log 是邏輯日誌，同步過程僅是單純的**循序寫入 (Sequential Write)**，速度極快。若直接寫入 InnoDB，需處理 B+Tree 分裂、鎖競爭等開銷 (Random Write)，會嚴重拖慢拉取速度。
    *   **錯誤隔離**：將「拉取」與「回放」解耦。即使 SQL 執行失敗，Relay Log 依然存在，可以修正後重試，無需重新向 Master 請求資料。

2.  **雙執行緒模型 (Dual-Thread Model)**：
    透過 Relay Log，MySQL 將同步工作拆解為兩個獨立的 Thread：
    *   **Replica I/O Thread**：負責 **[拉取]**。連上 Master -> 讀取 Binlog -> 寫入 Relay Log。
    *   **Replica SQL Thread**：負責 **[回放]**。讀取 Relay Log -> 解析 SQL -> 寫入 Storage Engine。


> [!NOTE]
> **什麼是 Relay Log？**
> Relay Log (中繼日誌) 是 Slave Server 上的一組日誌檔案，其格式與 Binlog 完全相同。
> *   **儲存位置**：預設位於 Data Directory (`datadir`)，檔名格式通常為 `{hostname}-relay-bin.xxxxxx` (其中 `{hostname}` 為 Slave Server 的 OS 主機名稱)。
> *   **生命週期**：I/O Thread 負責寫入，SQL Thread 執行完畢後會自動刪除 (Purge)，通常不需手動維護。
> *   **內容**：包含從 Master 拉取過來的原始 Binlog Events。

### 同步效能優化 (I/O Thread vs SQL Thread)

Replica I/O & SQL Thread 作為同步核心，優化方向取決於其瓶頸：

| 組件 (Thread) | 工作特性 | 瓶頸 (Bottleneck) | 優化策略 (Optimization) |
| :--- | :--- | :--- | :--- |
| **I/O Thread** | **單執行緒 (Serial)**<br>需確保順序與 Master 一致，無法平行化。 | **網路吞吐量 (Network)**<br>跨網路傳輸大量資料。 | **Binlog 壓縮** (`binlog_transaction_compression`)<br>透過 CPU 換取頻寬，減少網路傳輸量 (MySQL 8.0.20+)。 |
| **SQL Thread** | **複雜運算 (Complex)**<br>需解析 SQL、檢查約束、更新索引。 | **CPU & Disk IOPS**<br>通常是同步延遲 (Lag) 的主因。 | <span style="color: orange">**MTS (Multi-Threaded Slave)**<br>將回放工作平行化，開啟多個 SQL Thread 同時寫入。 </span> |

要啟用 **MTS (Multi-Threaded Slave)** 並行回放，核心挑戰在於 **「如何識別資料依賴 (Data Dependency)」**。若兩個 Transaction 修改同一筆資料，執行順序將決定最終結果 (Race Condition)。

**1. 有依賴關係 (Dependency Exists) - 必須循序執行**

```sql
--- Transaction A: 將 User 1 狀態設為 2
UPDATE users SET status = 2 WHERE user_id = 1;

--- Transaction B: 將所有狀態為 2 的用戶改成 1
UPDATE users SET status = 1 WHERE status = 2;
```

*   **若順序為 A -> B**：User 1 最終狀態為 `1` (先變 2，隨後被 B 的條件命中變成 1)。
*   **若順序為 B -> A**：User 1 最終狀態為 `2` (B 先執行時 User 1 非 2，不受影響；隨後 A 執行將其設為 2)。
> [!WARNING] 
> 由於執行順序會嚴重影響資料一致性，這類 Transaction **禁止並行 (Must be Serialized)**。

**2. 無依賴關係 (Independent) - 可以並行執行**

最簡單的判別方式是檢查是否跨不同資料庫 (**Per-Database**)，因為不同 DB 的資料實體是隔離的：

```sql
--- Transaction A (操作 DB_A)
UPDATE db_a.users SET status = 2 WHERE user_id = 1;

--- Transaction B (操作 DB_B)
UPDATE db_b.users SET status = 1 WHERE status = 2;
```

*   無論誰先執行，`db_a` 與 `db_b` 的最終結果都互不影響。
*   這類操作 **可以安全並行 (Parallel Safe)**。

基於上述「跨 DB 必無依賴」的邏輯，MySQL 5.6 引入了 **Per-Database Replication** (基於庫的並行複製)。

> [!WARNING] 
> 但此技術的致命傷在於**並行顆粒度 (Granularity) 過大**——它僅以「資料庫」為單位進行隔離：
> *   **不同 Database** 的交易 -> **可並行**。
> *   **同一個 Database** 的交易 -> **必須循序執行 (Serialized)**。
> 由於現代應用架構多採 **單一資料庫 (Single Database)** 模式，這導致幾乎所有交易都擠在同一個佇列中，根本無法享受到並行加速的紅利，導致該優化實用性不足。

因此 MySQL 5.7 推出了基於 **Logical Clock** 的 **MTS (Multi-Threaded Slave)** 技術，將並行判斷的顆粒度縮小至 **Transaction** 等級：

### Logical Clock Replication (MySQL 5.7+)


> [!NOTE]
> 
> 只要能夠在 Master 上 [同時進入 Commit 階段] 的 Transactions，代表彼此沒有 Lock 競爭 (否則會被 Block 住)，因此在 Slave 上也可以 [並行回放]。

這個機制利用了 Master 的 **Group Commit** 行為來標記並行性。具體實作上，MySQL 在 Binlog Event 中加入了兩個欄位來追蹤依賴關係：
1.  **`sequence_number`**：交易的流水號 (遞增)。
2.  **`last_committed`**：該交易提交時，系統中 **「最新已完成」** 的交易流水號。


![alt text](<imgs/Screenshot 2026-01-19 at 7.55.21 PM.png>)


**判斷依據：**
如上圖，多筆 Transaction 若擁有**相同的 `last_committed`**，代表它們是在同一個時間區間內執行並準備提交的。這意味著：
*   它們之間沒有鎖衝突 (Lock Contention)。
*   Slave SQL Threads 可以安全地並行執行這些 Transaction，最後再依序 Commit。




## MySQL 如何架設高可用的 Master-Slave 架構？(ProxySQL & Orchestrator)

架設高可用的 MySQL Cluster 不只要讓多台 Slave 去接收 Master 的 Binlog 同步資料，還要做到當 Master **Crush** 時，Slave 能自動接替成為 Master 同時 Client 送出的寫入請求也能自動轉換。

那麼要如何建立多個 Slave 去監聽 Master 的 Binlog？

### 首先架設 Master & Slave 需要設定這些參數：

| 參數 (Parameter) | 說明 (Description) |
| :--- | :--- |
| `server-id` | **Server 唯一識別碼**<br>Cluster 成員識別身份用 (避免循環複製)。需固定為唯一整數，不建議綁定 IP/Hostname。 |
| `log-bin` | **開啟 Binlog 功能**<br>啟用後才能紀錄變更並同步給 Slave。 |
| `log-bin-basename` | **Binlog 檔名路徑前綴** (預設 `binlog`)<br>可指定路徑 (e.g. `/log_disk/binlog`) 將 Log 存於獨立硬碟，避免與資料讀寫搶 IO。 |
| `binlog-format` | **Binlog 紀錄格式**<br>建議設為 `ROW` 以確保資料一致性。 |
| `gtid-mode` | **啟用 GTID 模式**<br>開啟全域交易 ID 用於追蹤同步進度 (`ON`)。 |
| `enforce_gtid_consistency` | **強制 GTID 一致性**<br>禁止執行非 GTID-Safe 的語句 (如 `CREATE TABLE ... SELECT`)，確保同步安全性。 |
| `log_replica_updates` | **Slave 寫入 Binlog** (舊版為 `log_slave_updates`)<br>讓 Slave 重播 Master 資料時也寫入自己的 Binlog。用於串接 (A->B->C) 或 Slave 晉升為 Master 後供其他節點同步。 |
| `binlog_expire_logs_seconds` | **Binlog 保留時間 (秒)**<br>自動清除舊 Log，避免塞爆硬碟空間。 |
| `bind-address` | **監聽 IP 位址**<br>設為 `0.0.0.0` 允許外部 (Slave) 連線，預設 `127.0.0.1` 只能本機連線。 |

而 **Slave** 的設定跟 Master 一樣，只是多了 read-only 確保他不能處理寫入請求。

```yaml
services:  
   mysql1:  
    container_name: mysql1  
    image: mysql/mysql-server:8.0  
    platform: linux/amd64  
    networks:  
        - mysql_cluster  
    environment:  
        MYSQL_ROOT_PASSWORD: secret  
        MYSQL_ROOT_HOST: "%"  
        MYSQL_DATABASE: local_test  
        MYSQL_USER: test  
        MYSQL_PASSWORD: test  
    volumes:  
        - ./data/mysql1:/var/lib/mysql  
    ports:  
        - 127.0.0.1:3306:3306  
    command:  
    [  
        "--server-id=1",  
        "--log-bin=mysql1-bin",  
        "--binlog-format=ROW",  
        "--gtid-mode=ON",  
        "--enforce-gtid-consistency=ON",  
        "--log-replica-updates=ON",  
        "--binlog-expire-logs-seconds=604800",  
    ]  
  
   mysql2:  
    container_name: mysql2  
    image: mysql/mysql-server:8.0  
    platform: linux/amd64  
    networks:  
        - mysql_cluster  
    environment:  
        MYSQL_ROOT_PASSWORD: secret  
        MYSQL_ROOT_HOST: "%"  
        MYSQL_DATABASE: local_test  
        MYSQL_USER: test  
        MYSQL_PASSWORD: test  
    volumes:  
        - ./data/mysql2:/var/lib/mysql  
    ports:  
        - 127.0.0.1:3307:3307  
    command:  
    [  
        "--read_only=ON",  
        "--port=3307",  
        "--server-id=2",  
        "--log-bin=mysql2-bin",  
        "--binlog-format=ROW",  
        "--gtid-mode=ON",  
        "--enforce-gtid-consistency=ON",  
        "--log-replica-updates=ON",  
        "--binlog-expire-logs-seconds=604800",  
    ]  
  
networks:  
    mysql_cluster:  
    driver: bridge
```

執行 `docker compose -f ./mysql-cluster.yaml up -d` 啟動 server 後，需要進入 slave db 設定 Master DB 連線資訊：

```sql
CHANGE REPLICATION SOURCE TO  
SOURCE_HOST = 'mysql1',  
SOURCE_PORT = 3306,  
SOURCE_USER = 'root',  
SOURCE_PASSWORD = 'secret',  
SOURCE_AUTO_POSITION = 1;  
  
START REPLICA;  
```

## 驗證同步狀態

執行以下指令檢查同步狀態：

```sql
SHOW SLAVE STATUS\G;
```

重點檢查以下兩個欄位必須為 `Yes`：
*   **`Slave_IO_Running`**: 連線正常，正在拉取 Binlog。
*   **`Slave_SQL_Running`**: 執行正常，正在回放 SQL。

---

## 如何在既有 Cluster 中加入新 Slave？ (Database Backup & Restore)

建立全新的 Cluster 很簡單，但要在一個「已經運行很久、資料量很大」的 Cluster 中加入一台新的 Slave，挑戰就來了：
**Master 的 Binlog 可能早就過期被刪除了 (Expired)**，新 Slave 無法從第一筆 Binlog 開始追。

這時我們必須先將 Master 目前的資料 **「搬運」** 到 Slave 上，讓 Slave 擁有一個近期的基準點 (Snapshot)，再利用 Binlog 追趕剩下的進度。
主要有兩種搬運方式：**邏輯備份** 與 **物理備份**。

### 方法 1: 邏輯備份 (Logical Backup) - mysqldump

**邏輯備份**是將 DB 中的資料全部轉譯成 `INSERT` SQL 語法。

#### 備份指令 (Master 端)

```bash
mysqldump -h 127.0.0.1 -u root -p \
  --single-transaction \   # (關鍵) 使用 Snapshot Read，確保備份過程中不鎖表且資料一致
  --quick \                # 逐行讀取，避免一次將大量資料載入記憶體 (OOM)
  --master-data=2 \        # (關鍵) 在輸出檔中註記當下的 Binlog Position/GTID (作為同步起點)
  --set-gtid-purged=ON \   # 輸出 GTID_PURGED 設定，告訴 Slave 這些 GTID 已包含在 Dump 檔中
  --triggers --routines --events \
  local_test > backup.sql
```

#### 還原與設定 (Slave 端)

```sql
-- 1. 暫時關閉 Binlog 寫入，避免還原過程產生大量無意義 Log
SET SQL_LOG_BIN=0;

-- 2. 匯入資料 (這會執行大量的 INSERT)
SOURCE /path/to/backup.sql;

-- 3. 匯入完成後，backup.sql 內含的 SET @@GLOBAL.GTID_PURGED='...' 
--    已經告訴 Slave 哪些 GTID 不需要再跟 Master 要了。

SET SQL_LOG_BIN=1;

-- 4. 啟動同步
START SLAVE;
```

#### 優缺點分析
| 特性 | 說明 |
| :--- | :--- |
| **優點** | **相容性高** (可跨版本、跨平台)、**可讀性高** (就是文字檔)。 |
| **缺點** | **速度慢** (需由 CPU 解析 SQL)、**還原久** (需重建 Index)、**消耗 Master 資源**。 |

---

### 方法 2: 物理備份 (Physical Backup) - Percona XtraBackup

**物理備份**是直接複製 MySQL 的 data 檔案 (`.ibd` files)。但因為複製過程中資料仍在變動，直接 `cp` 複製出來的檔案是**損毀的 (Inconsistent/Corrupt)**。
`Percona XtraBackup` 解決了這個問題：它在複製檔案的同時，會監控並複製 `Redo Log`。

#### 步驟 1: 備份 (Backup)
從 Master 複製檔案，並紀錄備份期間產生的 Redo Log。

```bash
docker run --rm \
  --volumes-from mysql_master \
  -v ./backup:/backup \
  percona/percona-xtrabackup:8.0 \
  xtrabackup --backup \
  --target-dir=/backup \
  --user=root --password=secret --host=mysql_master
```

#### 步驟 2: 準備 (Prepare) - **最關鍵的一步**
利用備份期間抓到的 `Redo Log`，對剛剛複製下來的檔案進行 **Crash Recovery**，將檔案修復成一致的狀態。

```bash
docker run --rm \
  -v ./backup:/backup \
  percona/percona-xtrabackup:8.0 \
  xtrabackup --prepare \  # (關鍵) 重放 Redo Log
  --target-dir=/backup
```

完工後，直接將 `/backup` 資料夾塞回 Slave 的 `datadir` 即可啟動。

#### 優缺點分析
| 特性 | 說明 |
| :--- | :--- |
| **優點** | **速度極快** (直接硬碟 IO 複製)、**還原快** (無需重建 Index)、**對 Master 影響小**。 |
| **缺點** | **限制嚴格** (OS、MySQL 版本需一致)、**檔案巨大**。 |

---

## 如何實現自動故障轉移 (Auto Failover)？ - Orchestrator

Master 即使再強壯也可能掛掉。我們需要一個「管理員」來監控 Cluster，當 Master 死掉時，自動選出一台最強的 Slave 晉升為新 Master，並讓其他 Slave 改跟隨新 Master。
這個工具就是 **[Orchestrator](https://github.com/openark/orchestrator)**。

![Orchestrator Topology](https://hackmd.io/_uploads/HJFndK94bg.png)

### 1. 核心配置 (Configuration)

`orchestrator.conf.json` 重點參數解析：

#### 基礎連線
Orchestrator 需要連線到 MySQL Cluster 進行監控 (`MySQLTopologyUser`)，也需要一個自己的 DB 存狀態 (`MySQLOrchestrator*`)。
```json
{
  "MySQLTopologyUser": "orchestrator_monitor",
  "MySQLTopologyPassword": "secret",
  "MySQLOrchestratorHost": "orchestrator_db",
  "MySQLOrchestratorDatabase": "orchestrator",
  // ...
}
```

#### 服務發現 (Service Discovery)
Orchestrator 如何找到所有的 Slave？
*   **方法 A (推薦)**: `DiscoverByShowSlaveHosts: true`。利用 MySQL `SHOW SLAVE HOSTS` 指令 (需配置 `report_host`)。
*   **方法 B**: 掃描 Processlist，尋找連接中的 Binlog Dump Thread。

#### 拓樸解析 (Topology Resolution)
在 Docker 或複雜網路環境中，Hostname 可能無法解析。
*   **`HostnameResolveMethod`**: 如何將 Hostname 轉為 IP (e.g., `default`, `env:HOSTNAME`).
*   **`MySQLHostnameResolveMethod`**: 當 Slave 回報 Master 是 "mysql1" 時，Orchestrator 該如何理解 "mysql1" 是誰。

### 2. 故障轉移機制 (Failover Logic)
當 Orchestrator 偵測到 Master 連不上時，會觸發 Failover 分析：

1.  **DeadMaster**: Master 連不上。
2.  **SlavesDangling**: 該 Master 底下的 Slaves 也回報連不到 Master (確認不是 Orchestrator 自己的網路問題)。

![Screenshot 2026-01-06 at 21.11.06](https://hackmd.io/_uploads/HkN0Yt5EZe.png)
![Screenshot 2026-01-06 at 21.11.13](https://hackmd.io/_uploads/SJOAFFcNbg.png)

**關鍵參數：**
*   **`ApplyMySQLPromotionAfterMasterFailover`**: `true`。允許 Orchestrator 自動執行寫入動作 (提拔 Slave)。
*   **`PreventCrossDataCenterMasterFailover`**: 避免選到跨機房的 Slave (延遲考量)。
*   **`FailMasterPromotionIfSQLThreadNotUpToDate`**: 若 Slave 資料落後太多，禁止提拔 (避免資料遺失)。

### 3. Hooks (Webhooks/Scripts)
Failover 前後可以執行 Script 通知 Slack 或呼叫 API。
```json
"PostFailoverProcesses": [
  "echo 'Recovered from {failureType} on {failureCluster}. New Master: {successorHost}' >> /tmp/recovery.log"
],
```




## Client 如何無痛切換？ - ProxySQL (Traffic Routing)

Orchestrator 換了 Master，但 Application 的 config 檔寫的還是舊 IP 怎麼辦？
我們需要在 App 與 DB 之間加一層 **Layer 7 Load Balancer** —— **[ProxySQL](https://proxysql.com/)**。

App 只需連線到 ProxySQL，由 ProxySQL 負責將 SQL 轉發給當下的 Master 或 Slave。

### ProxySQL 核心功能
1.  **Read/Write Split (讀寫分離)**: 自動將 `SELECT` 導向 Slave，`INSERT/UPDATE` 導向 Master。
2.  **Connection Pooling**: 復用連線，保護後端 DB 不被瞬間連線衝垮。
3.  **Failover Awareness**: 當 Orchestrator 切換 Master 後，ProxySQL 會自動偵測 `read_only` 狀態，調整路由。

### 基礎配置範例

ProxySQL 的設定是動態的 (存於 SQLite)，以下為核心概念配置 (類似 `proxysql.cnf`)：

#### 1. 定義 Hostgroups (群組)
將 DB 機器分組，例如：
*   **HG 1 (Writer)**: 放 Master。
*   **HG 2 (Reader)**: 放 Slaves。

```cpp
// 定義 backend servers
mysql_servers =
(
    {
        address="mysql1"
        port=3306
        hostgroup=1
        max_connections=200
    },
    {
        address="mysql2"
        port=3306
        hostgroup=1  // 備選 Writer
        max_connections=200
    }
)
```

> ProxySQL 會監控 `read_only` 變數。若 `mysql1` 變成 `read_only=ON` (變 Slave)，ProxySQL 會自動把它移出 HG 1。

#### 2. 定義路由規則 (Query Rules)
利用 Regex 決定 SQL 去哪。

```cpp
mysql_query_rules =
(
    {
        // Rule 1: SELECT ... FOR UPDATE 必須去 Master (避免讀到舊資料)
        rule_id=1
        active=1
        match_pattern="^SELECT .* FOR UPDATE$"
        destination_hostgroup=1
        apply=1
    },
    {
        // Rule 2: 所有讀取 (SELECT) -> 去 HG 2 (Reader)
        rule_id=2
        active=1
        match_pattern="^SELECT"
        destination_hostgroup=2
        apply=1
    },
    {
        // Rule 3: 所有寫入 (INSERT/UPDATE/DELETE) -> 去 HG 1 (Master)
        rule_id=3
        active=1
        match_pattern="^INSERT|^UPDATE|^DELETE"
        destination_hostgroup=1
        apply=1
    }
)
```

透過 Orchestrator 處理 **"Server 端"** 的 HA，加上 ProxySQL 處理 **"Client 端"** 的路由，就構成了一套完整的高可用 MySQL Cluster 架構。

詳細設定可參考：https://ithelp.ithome.com.tw/articles/10381791