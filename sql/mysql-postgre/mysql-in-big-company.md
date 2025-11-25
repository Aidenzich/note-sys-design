# MySQL 在大公司中的應用

MySQL 不僅支撐了 Facebook, Uber, GitHub, Shopee 等大公司的核心業務，甚至可以說，它是這幾家科技巨頭在發展過程中最重要的資料庫基石之一。

不過，他們使用的 MySQL 通常不是我們一般下載的「標準版」，而是經過高度客製化、架構優化或搭配特殊中間件（Middleware）的版本。

以下針對這四家公司與 MySQL 的關係進行詳細解析：
### 1. Facebook (Meta): MySQL 是社交圖譜的引擎
Facebook 是全世界最大的 MySQL 使用者之一。

* **核心用途：** 儲存著名的 **Social Graph（社交圖譜）**，包括你的貼文、按讚、留言和好友關係。
* **技術挑戰：** 當資料量達到 PB（Petabyte）等級時，標準的 InnoDB 引擎會遇到寫入瓶頸和空間效率問題。
* **解決方案：** Facebook 為了 MySQL 開發了全新的儲存引擎 **MyRocks**（基於 RocksDB）。
    * MyRocks 大幅壓縮了資料體積，並優化了寫入效能，讓他們能在不購買兩倍硬碟的情況下，支撐全球數十億用戶的活動。
    * 他們採用了大規模的 **Sharding（分片）** 架構，管理著數萬台 MySQL 伺服器。

### 2. Uber: 著名的「去 Postgres 轉 MySQL」遷移
Uber 在早期其實是使用 PostgreSQL，但在 2013-2016 年經歷了高速成長後，他們做出了一個在技術圈非常有名的決定：**從 Postgres 遷移到 MySQL**。

* **核心用途：** 叫車行程數據、司機與乘客的媒合資訊。
* **遷移原因：** 當時 Uber 指出 Postgres 在他們的使用場景下（高頻更新）會產生嚴重的 Write Amplification（寫入放大）問題，且當時 MySQL 的 Replication（複製）機制對他們來說更穩定。
* **架構特點：** Uber 在 MySQL 之上建立了一個名為 **Schemaless** 的分片層。這讓他們把 MySQL 當作一個底層的 Key-Value 儲存庫來用，既享受了 MySQL 的穩定性，又避開了頻繁更改 Table Schema 的痛苦。

### 3. GitHub (早期與現在): Rails 與 MySQL 的經典組合
GitHub 是基於 Ruby on Rails 框架構建的，而 Rails 社群與 MySQL 的關係非常緊密。

* **核心用途：** 儲存所有的非 Git 資料，例如 Issue 追蹤、Pull Request 資訊、用戶資料、評論等（Git 的程式碼本身是存於硬碟的 Git 結構中，但元數據在 MySQL）。
* **技術演進：** GitHub 從早期的單體資料庫，一路演進到使用 **Vitess**（原本由 YouTube 開發的 MySQL 水平擴展系統）。
* **貢獻：** GitHub 的工程團隊非常強大，他們開源了著名的 **Orchestrator** 工具，專門用來管理 MySQL 的高可用性（HA）和故障轉移（Failover），這套工具後來成為業界標準。

### 4. Shopee (蝦皮): 電商的高併發交易
Shopee 作為東南亞與台灣最大的電商平台，必須處理極高併發的「秒殺」搶購和龐大的訂單流。

* **核心用途：** **交易系統（Transactional Systems）**。在電商領域，庫存扣減、訂單生成、金流紀錄必須具備 **ACID** 特性（原子性、一致性、隔離性、持久性），MySQL 的 InnoDB 引擎是這方面的黃金標準。
* **架構特點：** 為了應對「雙 11」、「9.9」這種大促銷，單機 MySQL 無法負荷。Shopee 大量使用 **Sharding（分庫分表）** 技術，以及近年來開始採用的 **TiDB**（這是一個兼容 MySQL 協議的分散式資料庫），讓應用程式以為自己在連線 MySQL，但底層卻能無限水平擴展。

---

### 為什麼這些巨頭都選 MySQL？

雖然現在有 MongoDB、Cassandra 等 NoSQL，以及 Spanner 等 NewSQL，但 MySQL 依然屹立不搖，主要原因有三：

1.  **極致的穩定性：** 經過幾十年的驗證，MySQL 不容易丟失資料（Data Loss），這對金融、訂單、用戶資料來說是底線。
2.  **成熟的生態系：** 當你遇到效能瓶頸時，全世界有現成的工具（如 ProxySQL, Vitess, Orchestrator）可以幫你解決，不需要重新造輪子。
3.  **人才好找：** 幾乎所有的後端工程師都懂 MySQL，這對快速擴張的公司來說至關重要。

### 總結

這四家公司不僅使用了 MySQL，還被「逼」出了 MySQL 的極限。他們證明了只要架構設計得當（利用 Sharding、快取、讀寫分離），MySQL 足以支撐世界級的流量。

### Appendix
- [Uber's Migration from PostgreSQL to MySQL](./uber-pg2mysql.md)