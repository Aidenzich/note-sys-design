# 系統設計筆記：MySQL (InnoDB) vs. PostgreSQL 深度解析

這份筆記探討兩大主流關聯式資料庫 (RDBMS) 的核心差異。MySQL 傾向於**輕便主義**（簡單、快、高併發讀取），而 PostgreSQL 傾向於偏向學術定義的**完美主義**（功能豐富、嚴謹、擴充性強）。


## 1\. 核心架構：世界觀的差異 (Architecture)

這是兩者最根本的區別，決定了資料儲存與連線的方式。

| 特性 | MySQL (InnoDB) | PostgreSQL |
| :--- | :--- | :--- |
| **架構設計** | **插拔式引擎 (Pluggable)**<br>MySQL 是 Server 層，下方可掛載不同引擎，目前 InnoDB 是絕對主流。 | **統一引擎 (Unified)**<br>單一且整合度極高的 Heap Storage 引擎，強調標準與擴充性。 |
| **物理儲存** | **Clustered Index (叢集索引)**<br>資料直接存在 Primary Key 的 B-Tree 葉節點上。<br>$\rightarrow$ **主鍵查詢極快**。 | **Heap Table (堆疊表)**<br>資料隨意堆在 Page (Heap) 裡，索引與資料分開（索引存的是指標）。<br>$\rightarrow$ **Secondary Index 查詢效率較一致**。 |
| **連接模型** | **Thread-based (執行緒)**<br>每個連線是一個 Thread。輕量，Context Switch 開銷低，適合超高併發短連線。 | **Process-based (進程)**<br>每個連線是一個 Process (Fork)。資源消耗較大，通常需搭配 Connection Pool (如 PgBouncer)。 |



## 2\. 併發控制：MVCC 實作機制 (Concurrency)

兩者都使用 Multiversion concurrency control (MVCC) 多版本併發控制，它是一種資料庫管理系統的併發控制協議，**當寫入發生時，不會直接修改舊數據，而是建立一個新的版本，舊版本會被保留。讀取時，系統會根據當前事務的隔離級別，從所有版本中選擇一個符合條件的歷史快照版本**。主要優點在於讀取和寫入操作可以並行執行，不會互相衝突，實現了「讀不阻塞寫，寫不阻塞讀」。 

> 註：在 NoSQL 如 Cassandra 中，節點內部的 LSM-Tree 其實就是 MVCC 的一種實現形式，並依賴物理時間戳的 LWW 解決節點間衝突；而在 DynamoDB 中，AWS 實務上已捨棄論文中的 Vector Clock，轉而採用 LWW 策略；至於 Google Spanner 等 NewSQL，則在節點間利用 Paxos + TrueTime (Commit Wait) 確保一致性，並在節點內維持 2PL + MVCC 的機制。

MySQL 和 PostgreSQL 在處理「更新 (Update)」和「舊版本資料」的方式截然不同。

| 比較維度 | MySQL (InnoDB) <br> Undo Log 模式 | PostgreSQL <br> Append Only 模式 |
| :--- | :--- | :--- |
| **更新邏輯**<br>(Update Logic) | **In-place Update (就地更新)**<br>1. 將舊資料搬移至 **Undo Log**。<br>2. 新資料直接寫入並覆蓋原位置。<br>3. 透過指標指回 Undo Log 供讀取舊版。 | **Append Only (追加寫入)**<br>1. **不動舊資料** (Tuple)，僅標記為「過期」。<br>2. 直接在「新位置」寫入一行新版資料。 |
| **垃圾回收**<br>(Cleanup) | **Purge Thread**<br>由背景執行緒默默清理 Undo Log，對主資料表干擾較小。 | **VACUUM (吸塵器)**<br>必須定期掃描表格，清理那些標記為過期的死元組 (Dead Tuples)。 |
| **主要優點**<br>(Pros) | **適合高頻更新 (Update Heavy)**<br>更新發生在 Undo Log，主表空間穩定，不易膨脹。 | **Rollback 極快**<br>因為舊資料還在原位，回滾時只需無視新資料即可。 |
| **主要缺點**<br>(Cons) | **Rollback 速度較慢**<br>因為需要從 Undo Log 把舊資料搬回來恢復原狀。 | **表格膨脹 (Table Bloat)**<br>若更新太快導致 VACUUM 來不及跑，廢棄資料會佔用大量空間，拖累掃描效能。 |

-----

## 3\. 查詢效能戰場：JOIN 與 優化器
為什麼在複雜查詢下 PostgreSQL 通常比 MySQL 快？

| 比較項目 | MySQL (InnoDB) | PostgreSQL |
| :--- | :--- | :--- |
| **A. JOIN 演算法**<br>(Join Algorithms) | **主要依賴 Nested Loop**<br>雖然 8.0 版本引入 Hash Join，但整合度與成熟度仍較低。 | **三種武器自動切換**<br>1. Nested Loop (小表)<br>2. **Hash Join** (大表殺手鐧)<br>3. Merge Join (排序資料) |
| **B. 並行查詢**<br>(Parallel Query) | **單執行緒 (Single-threaded)**<br>傳統上一條 SQL 通常只能使用一個 CPU 核心。 | **支援並行查詢**<br>可將一個大表 JOIN 的查詢拆解給多個 CPU 核心同時跑，速度快上數倍。 |
| **C. 查詢優化器**<br>(Optimizer) | **貪婪且簡單**<br>當 JOIN 層級過深 (5張表以上) 時，容易選錯執行路徑。 | **學院派 (GEQO)**<br>具備遺傳演算法，在極複雜關聯中也能找到最佳路徑。 |
| **反轉場景**<br>(MySQL 何時會贏？) | **簡單的主鍵查詢 (Simple OLTP)**<br>擁有 **Clustered Index** (查 ID 即拿資料) 加上輕量級 Thread model，速度往往快於 PG。 | **複雜分析 (OLAP)**<br>在此場景 PG 具備絕對優勢，但在簡單查詢中因 Heap 結構需多一次 I/O 而略遜。 |

-----

## 功能與開發體驗 (Features & Developer Experience)

這是 PostgreSQL 被稱為「現代資料庫首選」的主要原因，解決了 MySQL 長期以來的痛點。

| 比較項目 | MySQL 的痛點 | PostgreSQL 的解法 |
| :--- | :--- | :--- |
| **資料型別** | **弱型別/靜默錯誤**<br>早期容易發生資料截斷、無效日期而不報錯。 | **嚴格強型別**<br>不合規直接報錯 (Error)，保證資料品質絕對乾淨。 |
| **DDL (改表)** | **非交易式**<br>改表失敗無法 Rollback，可能導致表結構半殘；早期易鎖表。 | **Transactional DDL**<br>`DROP TABLE` 也可以 Rollback！加欄位通常只改 Metadata，瞬間完成。 |
| **JSON 能力** | 儲存為字串，索引依賴 Generated Column，功能有限。 | **JSONB + GIN 索引**<br>二進位儲存，可直接索引內部欄位，效能直逼 MongoDB。 |
| **GIS (地圖)** | 功能僅堪用，運算較弱。 | **PostGIS**<br>地表最強開源 GIS 引擎，LBS/地圖應用的唯一選擇。 |
| **複製機制** | **Binlog (Logical)**<br>傳送 SQL 語句，可能導致主從數據不一致。 | **WAL (Physical)**<br>串流複製底層 Page 變更，保證物理層級的一致性。  |
> 註：在 Uber 案例之後（Postgres 10 以後），PG 官方其實已經原生支援 Logical Replication，雖然物理層面（Heap + Index）的寫入放大問題依然存在，但複製層面的靈活性已經大幅提升



## 對 MySQL 的批判
以下是業界普遍對 MySQL 的批判，以及 PostgreSQL 如何解決這些問題：

| 缺陷面向 | MySQL 的痛點 | PostgreSQL 的解法 |
| :--- | :--- | :--- |
| **資料嚴謹度** | 容易發生靜默截斷、無效日期 (需依賴 Mode 設定，在 5.7/8.0 透過 `STRICT_TRANS_TABLES` 改善很多)。 | **嚴格強型別**，不合規直接報錯，保證資料品質。 |
| **修改結構 (DDL)** | 如果你執行 `ALTER TABLE` 跑到一半失敗了，**它無法 Rollback**。你的表會卡在一個半殘的狀態。早期版本加一個欄位需要鎖整張表（Lock Table），雖然後來有了 Online DDL (Instant Add Column)，仍有較高的鎖表風險。 | **Transactional DDL** (可 Rollback)，加欄位瞬間完成。PG 加欄位通常只是修改 Metadata，幾乎是瞬間完成，不會鎖死讀寫。 |
| **SQL 能力** | 子查詢優化較弱，分析型 SQL 效能差。 | **優化器強大**，擅長處理 Subquery, Window Function, CTE。 |
| **NoSQL 能力** | JSON 支援一般，GIS 功能堪用。 | **JSONB + GIN 索引** (類 MongoDB)，**PostGIS** (GIS 王者)。 |
| **鎖機制** | 容易出現 Gap Lock 導致的死鎖 (Deadlock)。 | 使用 MVCC 的 Snapshot Isolation，讀寫完全不衝突，鎖競爭較少。 |
| **複製 (Replication) 的機制：邏輯 vs. 物理** | **Binlog (Logical Replication):** MySQL 的複製主要是傳送 SQL 語句（或 Row 變更）。這雖然靈活，但容易出現「主從不一致」的情況（例如主庫執行成功，從庫因為某些原因執行失敗）。 | **WAL (Physical Replication):** PG 的串流複製 (Streaming Replication) 是傳送底層的硬碟 Page 變更 (WAL)。這保證了 Slave 跟 Master 在物理層面上是**一模一樣**的 (Byte-for-Byte Consistency)，資料可靠性極高。 |
| **UUID/GUID 寫入** | 由於 MySQL 是 Clustered Index，若使用 UUID 這種亂序 ID 作為主鍵，會導致嚴重的 Page Splitting 和寫入效能下降（因為要一直插隊），需要手動透過`UUID_TO_BIN(..., 1)`或改用 snowflake/ULID 解決。在 InnoDB 中，表就是索引，索引就是表 (Clustered Index)。 這意味著，資料行 (Row Data) 是直接儲存在 B+Tree 的葉子節點 (Leaf Nodes) 裡的。 | PG 的 Heap 結構對 UUID 的寫入相對友善：因為 Table 是 Heap 與 Index 是分開的，雖然Index 也會發生 Page Split (隨機I/O)，但這個 B-Tree 節點裡存的只有 (UUID, CTID 指標)，而不包含整行資料（例如你的 JSON 欄位、TEXT 欄位都不在這裡），因此寫入效能影響較小。 而為了完全解決 Page Split 的問題， PG 社群的標準答案是「直接改用有序ID」如 UUID v7 |




### 為什麼 MySQL 還存在？
因為 PG 也有它的代價：
1.  **VACUUM 機制:** PG 需要定期清理死資料，如果配置不當，表會膨脹 (Bloat)，導致效能下降。MySQL 的 Undo Log 機制比較沒有這個煩惱。

    | 場景特徵 | 具體例子 | PG 缺陷 | MySQL 的優勢|
    | :--- | :--- | :--- | :--- |
    | 高頻更新單一欄位 | 計數器、庫存、積分 | 產生大量 Dead Tuples，且容易導致索引膨脹。 | In-place Update，不產生垃圾。|
    | 高頻插入與刪除 | 任務隊列 (Queue)、Log 表 | 檔案尾端不斷增長，前端空間回收不及，掃描變慢。 | 空間重用率高，B+Tree 自動平衡。|
    | 混合長短事務 | 在 OLTP 系統上跑 OLAP 報表 | 長事務會「卡住」垃圾回收，導致短時間內垃圾暴增。 | Undo Log 即使變大，也不影響主表查詢速度。|
2.  **連線成本:** PG 每個連線是一個 Process，記憶體消耗大。在極高併發（例如秒殺、搶票，瞬間湧入 5 萬個連線）的情況下，MySQL 的 Context Switch 負擔較輕，能夠維持較高的吞吐量。
3.  **生態系極廣:** MySQL 雖然有缺點，但全世界的工程師都會用，工具最多，雲端支援最便宜。


## 總結與選型決策 (Decision Matrix)

| 比較維度 | MySQL (InnoDB) | PostgreSQL |
| :--- | :--- | :--- |
| **核心特質** | **實用導向**<br>結構簡單、耐操、零件好找（工程師多）、維修便宜。 | **科技導向**<br>功能多（JSON/GIS）、底盤紮實（學院派）。 |
| **擅長領域** | **跑市區 (簡單 CRUD)**<br>載人能力強（高併發連線）。 | **全地形 (複雜資料)**<br>自動駕駛（強大優化器），但車身較重（Process model），保養需專業（VACUUM）。 |
| **業務類型** | **超高併發簡單讀寫**<br>如：搶票、秒殺、計數器。 | **標準 SaaS / 新創專案**<br>適合未來需求不確定、需要快速迭代的場景。 |
| **資料特性** | **遊戲伺服器 / Key-Value 應用**<br>簡單的存取模式，極致的速度。 | **複雜結構 / 地理資訊**<br>需要處理 JSON, Array 或 GIS (PostGIS)。 |
| **運算需求** | **極端 Update Heavy**<br>頻繁更新同一行資料（避免 PG 的 Vacuum 膨脹問題）。 | **後台報表 / OLAP 分析**<br>涉及複雜 JOIN 查詢、極度重視資料正確性 (Strict Constraints)。 |
| **團隊因素** | **團隊極度熟悉 MySQL**<br>擁有豐富的維運與調校經驗。 | **需要強大的擴充性**<br>預期未來會有複雜的商業邏輯運算。 |

相較於 PostgreSQL，MySQL 的優勢在於來自 `undo log`, `thread-based` 所帶來的低空間使用率和低連線成本，使得 MySQL 確實相對的輕量與快速，但相對的，PostgreSQL 的 `append-only` 機制雖然帶來較高的空間使用率，但 Rollback 的效能更好，完善的 JOIN 優化器，更使得在複雜查詢或 OLAP 的使用環境效果更佳。另外，PostgreSQL 的 process-based 機制也讓一個 SQL 查詢可以使用多核心完成，卻也帶來需要而外配置 Proxy 如 PgBouncer, AWS RDS Proxy 的必要性， MySQL 則因為 thread 相對輕量，可以在沒配置 Proxy 的情況下，容納更多連線。


## Appendix
- [業界實務 MySQL vs PostgreSQL 選型](./mysql-in-big-company.md)
