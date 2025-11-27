# 系統設計筆記：MongoDB vs. Cassandra 深度解析

## 1\. 核心架構與儲存引擎 (Single Node Layer)

這是決定單機 (Single Node) 讀寫效能的物理基礎。

| 特性 | MongoDB (WiredTiger) | Cassandra (LSM-Tree) |
| :--- | :--- | :--- |
| **核心結構** | **B-Tree (B+ Tree)** | **LSM-Tree (Log-Structured Merge-tree)** |
| **寫入機制 (Write)** | **In-Place Update**<br>需維護樹結構平衡，記憶體不足時會觸發隨機 I/O (Page Fault)。 | **Append-Only**<br>順序寫入 (Sequential Write)，不需讀取舊資料，吞吐量極高。 |
| **讀取機制 (Read)** | **Direct Access**<br>透過索引直接指標跳轉 ($O(\log N)$)。延遲極低。 | **Merge & Compact**<br>需檢查 Memtable 與多個 SSTable，並處理合併邏輯 (Read Amplification)。 |
| **資源消耗** | **C++** (資源緊湊，啟動快) | **Java/JVM** (GC Pause 風險，Compaction 佔用 CPU/IO) |
| **單機結論** | **讀取勝、延遲低** | **寫入勝、吞吐量高** |

-----

## 2\. 巢狀資料處理 (Nested Data Layer)

這是決定開發靈活性與複雜查詢效能的關鍵。
在 MongoDB (Document DB) 的世界裡，Document (文檔) 是資料儲存的最小基本單位。
MongoDB 實際儲存時是用 BSON (二進位格式)。這讓它比純文字的 JSON 讀取更快、更省空間，並且支援更多資料類型 (如 Date, Binary Data)。 因為 BSON 格式允許資料庫快速跳到特定的欄位，而不需要讀取整份文件："Traversable" (可遍歷性)。**雖然 Cassandra 也可以存 JSON 字串，但它只是把它當成「文字」存, 必須反序列化 (Deserialize) 整個巢狀結構才能使用**


| 比較項目 | MongoDB (Document Store) | Cassandra (Wide Column Store) |
| :--- | :--- | :--- |
| **物理儲存格式** | **BSON (Traversable)**<br>儲存引擎「看得懂」內部結構。 | **Frozen Blob**<br>SSTable 將巢狀結構視為一筆二進位資料。 |
| **讀取特定欄位** | **支援 Projection**<br>可只讀取 `user.address.zip`，節省 IO 與頻寬。 | **不支援**<br>必須讀取整個 `user.address` 物件並**反序列化 (Deserialize)**，浪費 CPU/IO。 |
| **巢狀欄位索引** | **Multikey Index (B-Tree)**<br>標準 B-Tree 索引，支援高效查找。 | **Secondary Index (Hidden Table)**<br>效率差，查詢時需兩階段跳轉 (Index Table -\> Primary Key -\> SSTable)。 |
| **查詢模式** | **本地查詢極快**<br>即使 Broadcast，本地也是 $O(\log N)$。 | **Scatter-Gather 災難**<br>若無 Partition Key，需全叢集掃描 + 反序列化，效能極差。 |
| **結論** | **完勝** (First-class citizen) | **極不推薦** (應避免使用複雜巢狀結構) |

> **補充：Cassandra 的 Index 迷思**
>
>   * **Clustering Key:** 是實體排序 (Physical Sort)，範圍掃描極快，但必須依附於 Partition Key。
>   * **Secondary Index:** 是額外的隱藏表 (Hidden Table)，效率遠不如 MongoDB 的 B-Tree。

-----

## 3\. 分散式架構與擴展性 (Distributed Layer)

這是決定系統規模上限與維運難度的關鍵。

| 特性 | MongoDB (Sharding) | Cassandra (Peer-to-Peer) |
| :--- | :--- | :--- |
| **拓撲結構** | **Master-Slave (Replica Set)**<br>每個 Shard 只有一個 Primary 負責寫入。 | **Masterless (無主架構)**<br>所有節點均可寫入。 |
| **擴展能力** | **分片 (Sharding)**<br>需處理 Chunk Migration，若 Shard Key 選錯會有熱點問題。 | **線性擴展 (Linear Scalability)**<br>加機器即提升效能 (Token Ring 自動分配)。 |
| **一致性模型** | **CP / Strong Consistency**<br>預設強一致性。Primary 掛掉時會短暫無法寫入 (Election)。 | **AP / Eventual Consistency**<br>可用性優先。允許節點掛掉仍可寫入 (Hinted Handoff)。 |

-----

## 4\. 決策矩陣：依讀寫場景選擇 (Scenarios)

這是架構師做最終技術選型 (Trade-off) 的依據。

| 場景 | 推薦選擇 | 關鍵理由 (Key Factor) |
| :--- | :--- | :--- |
| **多讀少寫 (Read Heavy)** | **MongoDB** | B-Tree 索引強大，支援複雜查詢 (Ad-hoc Query)，開發靈活且讀取延遲低。 |
| **少讀多寫 (Write Heavy)** | **Cassandra** | LSM-Tree 的寫入吞吐量極高，適合 Log、IoT 數據、軌跡紀錄。 |
| **讀寫都重 + 簡單查詢** | **Cassandra** | 雖然讀寫都重，但因為查詢簡單 (By Key)，Cassandra 可避免 B-Tree 索引過大導致的 RAM 瓶頸，並提供線性擴展。 |
| **讀寫都重 + 複雜查詢** | **MongoDB (Sharded)** | 雖然維護 Sharding 成本高，但 Cassandra **無法有效處理**複雜查詢 (Secondary Index 效能差)。這時 MongoDB 較佳。 |


### 總結
#### 核心差異總表
| 比較維度 | Cassandra (Hidden Table / LSM-Tree) | MongoDB / MySQL (Native B-Tree) |
| :--- | :--- | :--- |
| **物理結構** | **LSM-Tree (Log-Structured)**<br>索引數據被視為 Log，分層儲存 (Memtable + SSTables)。 | **B+ Tree (Balanced Tree)**<br>索引數據維持嚴格的樹狀平衡結構，實時排序。 |
| **Write** | **極快 (Low Cost)** | **較慢 (High Cost)** |
| 寫入動作 | **Append-Only (追加)**<br>不管內容，直接寫入記憶體 (Memtable)。**當下即時排序 (Sorted)**，但不檢查舊資料 (No Read-Before-Write)。<br>當 Memtable 達到一定容量 (Threshold) 後，會直接 **Flush (刷寫)** 到硬碟，成為一個新的不可變更 (Immutable) 的 **SSTable**。 | **Read-Modify-Write (原地更新)**<br>需先找到樹的正確位置 $\rightarrow$ 載入 Page $\rightarrow$ 插入 $\rightarrow$ **實時排序**。 |
| I/O 行為 | **順序 I/O (Sequential)**<br>硬碟磁頭只需一直往下寫，不回頭。 | **隨機 I/O (Random)**<br>若 Index Page 不在 RAM 中，需讀取硬碟。且 Page 滿了會觸發 **Page Split**。 |
| 鎖競爭 (Locking) | **極低**<br>幾乎無鎖，適合高併發寫入狂灌。 | **較高**<br>維護樹結構時需對 Page 加鎖 (Latch)，高併發下易成為熱點。 |
| **Read** | **極慢 (High Cost)** | **極快 (Low Cost)** |
| 查詢動作 | **Scatter-Gather + Merge**<br>1. 廣播到所有節點 (Scatter)。<br>2. 查 Local Index 拿到 PK。<br>3. 查 SSTables (可能多層) 拿到 Blob。<br>4. **反序列化 Blob** 檢查內容。 | **Tree Traversal (樹遍歷)**<br>1. 根據 Shard Key 定位節點 (單播)。<br>2. 沿著 B-Tree 根節點 $\rightarrow$ 葉節點。<br>3. **直接指標跳轉**拿到資料。 |
| I/O 行為 | **讀取放大 (Read Amplification)**<br>查一次資料可能要讀多個 SSTable 檔案，且涉及大量反序列化 CPU 運算。 | **精確讀取 (Precise)**<br>時間複雜度穩定 $O(\log N)$。若 Index 在 RAM 內，幾乎無 Disk I/O。 |
| 巢狀資料支援 | **差 (Blob)**<br>必須讀出整個物件才能篩選，無法只讀部分欄位。 | **優 (Structure aware)**<br>支援 Projection，可只讀取巢狀內的特定欄位。 |
| **【最終結論】** | **寫入吞吐量的王者**<br>適合：IoT 數據、Log 收集、寫多讀少、簡單 Key-Value 查詢。 | **查詢靈活度的王者**<br>適合：使用者資料、電商產品、複雜報表、讀多寫少或讀寫均衡。 |



##### **Cassandra (LSM-Tree) 像是「忙碌的收發室」**

  * **寫入時 (收件)：** 信件來了？不管三七二十一，**先丟進箱子裡再說**！(貼個便利貼就完事，超快)。
  * **讀取時 (找件)：** 慘了，老闆要找「上個月來自 A 公司的信」。因為當初是用丟的，現在要**把所有箱子倒出來**，一封一封翻找、比對便利貼。(超慢，超累)。

##### **MongoDB / MySQL (B-Tree) 像是「強迫症的檔案室」**
  * **寫入時 (歸檔)：** 文件來了？**不行隨便丟！** 走到第 3 排櫃子，拉開第 5 個抽屜，找到「A」分類，按日期插進去。如果抽屜滿了，還要整理抽屜。(動作慢，規矩多)。
  * **讀取時 (調閱)：** 老闆要找「上個月來自 A 公司的信」。**直接走到**第 3 排第 5 抽屜，伸手一抓就是它。(超快，優雅)。





### Appendix. DynamoDB 比較

### 1\. 讀取速度 (Read Speed)
**分析：**
  * **DynamoDB vs. Cassandra：** **DynamoDB 勝。**
      * 如前所述，Cassandra 的 LSM-Tree 在讀取時有「讀取放大」與「合併」的成本，且延遲會抖動。
      * DynamoDB 是 B-Tree，讀取路徑是固定的 $O(\log N)$，且直接命中，非常穩定。
  * **DynamoDB vs. MongoDB：** **單次讀取 MongoDB 略快 (微乎其微)，但高併發下 DynamoDB 更穩。**
      * **通訊協定差異：**
          * **MongoDB:** 使用 TCP 長連線 (Persistent Connection)。應用程式與 DB 建立連線後，指令傳輸非常快。
          * **DynamoDB:** 使用 **HTTPS (REST API)**。每次請求都要經過 HTTP Handshake (雖然有 Keep-Alive)、Load Balancer、Request Router 層層轉發。這會增加 **1\~3ms** 的額外網路延遲。
      * **結論：** 如果是單測「讀取一筆資料」，MongoDB (1ms) 可能會險勝 DynamoDB (3\~5ms)。但在架構層級上，兩者都屬 B-Tree 陣營，讀取效能同級。

### 2\. 寫入速度 (Write Speed)
**分析：**

  * **DynamoDB vs. Cassandra：** **Cassandra 完勝。**
      * **物理限制：** DynamoDB 底層還是 B-Tree，寫入時要維護樹結構（雖然 AWS 用 SSD 暴力加速了）。
      * **機制限制：** Cassandra 是 LSM-Tree，採用 Append-Only 的方式，物理上 B-Tree 比不過 LSM-Tree 的寫入吞吐量。
  * **DynamoDB vs. MongoDB：** **DynamoDB 勝 (在大規模下)。**
      * **MongoDB (Sharding):**
          * 假設你有 3 個 Shard。
          * 寫入請求必須對應到這 3 台 Primary Node。
          * 如果流量暴衝，這 3 台 Primary 就是瓶頸。增加 Shard 需要時間搬移資料 (Chunk Migration)。
      * **DynamoDB (Partitioning):**
          * 雖然 DynamoDB 每個 Partition 內部有 Leader (Paxos)，但它的 Partition **極小** (早期是 10GB 一個，現在更靈活)。
          * 當你寫入量變大，DynamoDB 會把 Partition 分裂成 1000 個、10000 個。
          * **效果：** 雖然單機寫入速度 MongoDB 和 DynamoDB 差不多（都是 B-Tree），但 DynamoDB 透過 **「極細粒度的分散寫入」**，消除了 MongoDB 那種「單一 Primary 節點」的瓶頸。

### 3\. The Ranking
| 評比項目 | **最佳** | **次優** | **劣勢** | 理由 |
| :--- | :--- | :--- | :--- | :--- |
| **簡單讀取**<br>(Simple Read) | **MongoDB**<br>(TCP 連線快，B-Tree 直接命中) | **DynamoDB**<br>(B-Tree 穩定，但輸在 HTTP Overhead) | **Cassandra**<br>(輸在 LSM-Tree 合併成本) | B-Tree 陣營獲勝 |
| **簡單寫入**<br>(Simple Write) | **Cassandra**<br>(LSM-Tree 物理無敵) | **DynamoDB**<br>(輸在 B-Tree 維護，但贏在無限水平擴展) | **MongoDB**<br>(輸在 B-Tree 維護 + Sharding 瓶頸) | LSM-Tree 陣營獲勝 |
| **複雜查詢**<br>(Complex Query) | **MongoDB**<br>(Aggregations, Join, Flexible Index) | **Cassandra**<br>(雖弱，但比 DynamoDB 稍微好一點點點) | **DynamoDB**<br>(只能 Scan，極弱) | 功能性差異 |
| **擴展性/維運**<br>(Scalability) | **DynamoDB**<br>(全託管，睡覺也會自動擴展) | **Cassandra**<br>(線性擴展，但要自己加機器) | **MongoDB**<br>(分片維運最痛苦) | Serverless 優勢 |

### 補充：DynamoDB 是 Masterless 嗎？
* **Cassandra** 是真正的 **P2P Masterless/Leaderless** (Gossip Protocol)。
* **DynamoDB** 其實是 **Multi-Leader (Partition-based Leader)**。
  * 它沒有「一台」總 Master。
  * 但它成千上萬個小 Partition 裡，每一個 Partition 都有一個 **Leader** (用 Paxos 演算法選出來的)。


### 實例架構決策表 (Final Cheat Sheet)
| 需求特徵 | 首選資料庫 | 關鍵理由 | 典型場景 |
| :--- | :--- | :--- | :--- |
| **巢狀資料 + 靈活查詢** | **MongoDB** | BSON 結構 + 強大索引 (Multikey/Geo) + 靈活 Schema。 | CMS 內容管理、使用者資料 (User Profile)、產品型錄。 |
| **超高寫入 + 讀取簡單** | **Cassandra** | LSM-Tree 帶來的極致寫入吞吐量 + 線性擴展。 | 通訊軟體 (Line/WhatsApp) 訊息紀錄、IoT 感測器 Log、使用者軌跡。 |
| **高讀寫 + 極低延遲 + 波動大** | **DynamoDB** | 穩定的 B-Tree + 無限自動分片 + Serverless 免維運。 | 購物車、遊戲玩家存檔、即時競價、高併發計數器。 |
| **極致速度 + 資料量小** | **Redis** | 純記憶體運算 (In-memory)。 | 快取 (Cache)、排行榜 (Leaderboard)、Session (短期)。 |
| **複雜關聯 + 交易一致性** | **PostgreSQL / MySQL** | ACID 交易 + Join 能力 + 嚴謹約束。 | 金流訂單、銀行系統、企業 ERP。 |

**簡記：**
* 要 **靈活** $\rightarrow$ Mongo
* 要 **吞吐量** $\rightarrow$ Cassandra
* 要 **穩** (且有錢) $\rightarrow$ DynamoDB
* 要 **快** $\rightarrow$ Redis
* 要 **準** $\rightarrow$ SQL