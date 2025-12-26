# **Apache Kafka 深度核心技術剖析報告：從分散式日誌抽象到事件流平台的架構演進**

## **1\. 緒論：事件流與分散式日誌的典範轉移**

### **1.1 起源與設計哲學：解決 $N \times N$ 的複雜度難題**

`Apache Kafka` 的誕生並非源於對現有訊息佇列（Message Queue）的單純改良，而是源於 `LinkedIn` 工程團隊對資料整合架構的根本性反思。在 `Kafka` 出現之前，企業內部的資料管線（Pipeline）多採用點對點（Point-to-Point）的傳輸模式。隨著業務系統的擴張，應用程式與資料存儲系統之間的連結呈現爆炸性的 $N \times N$ 網狀結構，導致了資料傳輸鏈路脆弱、延遲不可控且難以維護 1。

傳統的訊息中間件（如 `ActiveMQ` 或 `RabbitMQ`）雖然解決了部分解耦問題，但其設計初衷多是針對「複雜路由」與「任務佇列」，且通常假設訊息在被消費後即從記憶體或磁碟中刪除（短暫性存儲）。這種設計在面對海量日誌資料（Log Data）或使用者行為追蹤（User Activity Tracking）時，<span style="color: red;">往往受限於吞吐量與持久化能力的不足 [3]。</span>

Kafka 的核心哲學在於引入了「分散式提交日誌」`Distributed Commit Log` 這一抽象概念。日誌（Log）是一個按時間順序追加（Append-Only）、完全有序且不可變的記錄序列。Kafka 將這一概念從資料庫內部的實作細節中提取出來，作為分散式系統的核心骨幹。這種設計帶來了兩個根本性的轉變：

1. **持久化與重放（Durability & Replayability）：** 不同於傳統佇列的「閱後即焚」，Kafka 將資料持久化儲存於磁碟，允許消費者根據自身需求，隨時重放歷史資料或僅處理最新資料 [2]。
2. **儲存與計算分離（Decoupling of Storage and Compute）：** Kafka 作為純粹的儲存層（Storage Layer），僅負責高效的讀寫；而狀態管理與業務邏輯則交由客戶端或上層的流處理框架（如 `Kafka Streams`, `Flink`）處理 [4]。

在 Kafka 出現之前，如果你有 5 個來源系統（Producers）和 5 個目標系統（Consumers），為了傳輸資料，你可能需要建立 $5 \times 5 = 25$ 條連線。每增加一個系統，複雜度是乘法成長。Kafka 模式確保所有的來源系統只對接 Kafka（寫入 Topic），所有的目標系統也只對接 Kafka（訂閱 Topic），連線數變成了 $5 \text{ Producer} + 5 \text{ Consumer} = 10$ 條。複雜度從 $O(N \times M)$ 降到了 $O(N + M)$。這在架構上被稱為解耦（Decoupling）。



### **1.2 核心資料模型：事件（Event）的結構化定義**

在 Kafka 的架構中，資料傳輸的基本單元被稱為 `事件 Event`或 `記錄 Record`。一個標準的 Kafka 事件並非是非結構化的二進位塊，而是包含四個關鍵組成部分，這些部分共同支撐了 Kafka 的語義功能 1：

| 組成部分 | 描述與作用 | 技術細節 |
| :---- | :---- | :---- |
| **Key** | 事件的鍵值，決定資料的分割（Partitioning）策略。 | 用於雜湊運算（如 Murmur2），確保同一實體（如 User\_ID: 101）的事件總是進入同一分區，保證局部有序性。 |
| **Value** | 實際的業務負載（Payload）。 | 通常使用 Avro、Protobuf 或 JSON 序列化。Kafka 對內容不可見（Opaque），僅視為位元組陣列。 |
| **Timestamp** | 事件發生的時間戳。 | 可分為 CreateTime（生產者創建時間）或 LogAppendTime（Broker 接收時間），對流處理的時間視窗計算至關重要。 |
| **Headers** | 選擇性的元資料鍵值對。 | 用於分散式追蹤（Tracing）、應用層路由或審計，不影響資料本身的序列化結構。 |


**2\. 儲存層物理實作深度解析：日誌段、索引與清理機制**

Kafka 的高效能很大程度上歸功於其對檔案系統結構的精巧設計。理解 Kafka 必須先深入其在磁碟上的物理佈局，這解釋了為何 Kafka 能在普通磁碟上跑出極高的吞吐量。

### **2.1 主題（Topic）與分區（Partition）的物理映射**

邏輯上，Topic 是一個分類事件的容器；但在物理層面上，Topic 被拆分為多個 Partition。Partition 是 Kafka 並行處理（Parallelism）與儲存的基本單位 5。在 Broker 的設定檔 log.dirs 指定的目錄下，每個 Partition 對應一個獨立的子目錄，命名規則通常為 `<topic_name>-<partition_index>`。

Partition 本質上是一個追加寫入（Append-Only）的日誌。為了避免單個檔案過大導致維護困難（如索引建立、檔案清理、與作業系統檔案控制代碼限制），Kafka 將 Partition 進一步在時間或大小維度上切分為多個「日誌段」（Log Segments）。

### **2.2 日誌段（Log Segment）的內部檔案結構**

每個 Log Segment 並非單一檔案，而是由一組共享相同「基底偏移量」（Base Offset）的檔案集合組成。Base Offset 是該 Segment 中第一條訊息的絕對 Offset。例如，若一個 Segment 從 Offset 1000 開始，其檔案名將為 00000000000000001000.log 6。

一個完整的 Log Segment 包含以下核心檔案：

1. **日誌數據檔案 (.log)：**  
   * 儲存實際的序列化訊息批次（Record Batches）。  
   * 寫入模式為嚴格的順序追加。  
   * 訊息在磁碟上的格式與在網路傳輸中的格式完全一致，這是實現 Zero-Copy 的前提。  
2. **偏移量索引檔案 (.index)：**  
   * **作用：** 映射「邏輯 Offset」到「物理檔案位置（Byte Position）」。  
   * **結構：** 每個索引項佔 8 bytes（4 bytes 相對 Offset \+ 4 bytes 物理位置）。相對 Offset 是指相對於 Base Offset 的增量，這節省了空間。  
3. **時間戳索引檔案 (.timeindex)：**  
   * **作用：** 映射「時間戳」到「邏輯 Offset」。  
   * **用途：** 支援基於時間的檢索（如「從昨天開始消費」）以及基於時間的日誌保留策略。每個索引項佔 12 bytes（8 bytes Timestamp \+ 4 bytes Relative Offset）8。  
4. **快照檔案 (.snapshot)：**  
   * **作用：** 用於儲存生產者狀態快照（Producer State Snapshot）。  
   * **背景：** 為了支援冪等性生產者（Idempotent Producer）和事務，Broker 需要記錄每個生產者 ID（PID）對應的最新序列號（Sequence Number）。這些狀態隨日誌滾動而持久化，確保重啟後能恢復去重能力 7。  
5. **Leader Epoch Checkpoint (leader-epoch-checkpoint)：**  
   * **作用：** 記錄每一代 Leader 任期（Epoch）開始時的起始 Offset。  
   * **重要性：** 用於解決副本資料一致性問題，特別是在 Leader 切換後的日誌截斷（Truncation）場景，防止資料丟失或不一致 8。

### **2.3 稀疏索引（Sparse Index）演算法與二分搜尋**

Kafka 的索引設計是其讀取效能的關鍵。不同於關聯式資料庫的 B+ Tree 索引（通常為每一行資料建立索引），Kafka 採用「稀疏索引」策略 10。

* **機制：** Kafka 不會為每一條訊息建立索引。相反，它由 log.index.interval.bytes 參數控制（預設 4KB），每當日誌檔案寫入累積達到一定位元組數時，才在 .index 檔案中追加一個索引項。  
* **權衡（Trade-off）：** 這種設計極大減少了索引檔案的大小，使其能夠完全常駐於記憶體（Page Cache）中，避免了查詢索引時的磁碟 I/O。代價是讀取時無法直接定位到確切的訊息，只能定位到「附近」。

**訊息查找流程（讀取 Offset X）：**

1. **記憶體定位 Segment：** Broker 在記憶體中維護一個 ConcurrentSkipListMap 或類似結構，儲存所有 Segment 的 Base Offset。使用二分搜尋法快速找到包含 Offset X 的 Segment（即 Base Offset \<= X 的最大 Segment）。  
2. **索引檔案二分搜尋：** 在對應的 .index 檔案中（已映射到記憶體），使用二分搜尋找到小於或等於 X 的最大索引項。假設找到的索引項指向物理位置 P。  
3. **日誌檔案順序掃描：** 從物理檔案的 P 位置開始，順序讀取並解析訊息，直到找到 Offset X。由於稀疏索引的間隔很小（預設 4KB），這一步的順序掃描非常快 7。

### **2.4 日誌清理策略：Compaction 與 Retention**

Kafka 的日誌不會無限增長，它提供了兩種清理策略（Cleanup Policy）6：

1. **Delete Policy（預設）：** 基於時間（log.retention.hours）或大小（log.retention.bytes）的保留策略。超過閾值的舊 Segment 會被標記刪除，隨後由檔案系統異步回收。  
2. **Compact Policy（日誌壓縮）：**  
   * **場景：** 適用於 Key-Value 類型的資料（如資料庫變更日誌 CDC）。  
   * **原理：** 系統保證對於任意一個 Key，日誌中至少保留該 Key 的**最新**一個 Value。舊的 Value 會被刪除。  
   * **實作：** Kafka 背景有一組 Cleaner Threads。它們讀取日誌段，建立一個 Key 到 Offset 的雜湊映射（SkimpyOffsetMap），然後複製日誌，剔除那些在映射中 Offset 較小的舊記錄，生成新的「已壓縮」Segment。這是一個重 I/O 的操作，因此通常會限制其 I/O 頻寬 6。  
   * **墓碑機制（Tombstone）：** 若一條訊息的 Value 為 null，則視為刪除標記（Tombstone）。Compact 過程會保留該標記一段時間（delete.retention.ms），以確保所有消費者都能感知到刪除事件，防止「殭屍資料」復活 13。



**3\. I/O 子系統與效能最佳化：速度的物理基礎**

儘管 Kafka 使用磁碟儲存，但其吞吐量卻能媲美甚至超越部分記憶體級的訊息佇列。這歸功於其對作業系統底層機制的極致利用，主要包括順序 I/O、Page Cache 與 Zero-Copy 技術。

### **3.1 順序 I/O (Sequential I/O) 的物理優勢**

Kafka 強制將所有寫入操作設計為順序追加（Append-only）。在儲存硬體層面，順序 I/O 與隨機 I/O 存在巨大的效能鴻溝 2。

* **機械硬碟 (HDD)：** 隨機 I/O 需要磁頭頻繁移動（Seek Time）和等待磁碟旋轉（Rotational Latency），這在物理上限制了 IOPS。而順序 I/O 僅需一次尋道，後續皆為資料傳輸，速度可達隨機寫入的 6,000 倍。  
* **固態硬碟 (SSD)：** 雖然 SSD 沒有機械結構，但順序寫入依然具有顯著優勢。隨機寫入會導致嚴重的寫入放大（Write Amplification）問題，增加垃圾回收（GC）負擔並縮短壽命。順序寫入則能最大化快閃記憶體的編程效率。

### **3.2 頁面快取（Page Cache）與寫後快取（Write-Back）**

Kafka 選擇不自己在 JVM Heap 中維護訊息快取，而是將快取管理權完全移交給作業系統的核心（Kernel）15。

#### **3.2.1 寫入路徑**

當 Kafka Broker 收到生產者的訊息時，它調用 write() 系統呼叫將資料寫入檔案。

1. **寫入 Page Cache：** 資料首先被寫入 OS 的 Page Cache（實體記憶體）。  
2. **標記髒頁（Dirty Page）：** 該記憶體頁面被標記為「髒」。此時，Kafka 的寫入操作在應用層面已經完成，延遲極低。  
3. **背景刷盤（Background Flush）：** OS 的背景執行緒（如 Linux 的 pdflush 或 flush 執行緒）負責根據核心參數（如 vm.dirty\_ratio, vm.dirty\_background\_ratio）將髒頁異步寫入物理磁碟。

**風險與權衡：** 這種機制意味著 Kafka 承認在作業系統崩潰（OS Crash）或斷電時可能丟失未刷盤的資料。Kafka 通過\*\*多副本複製（Replication）\*\*而非強制的物理刷盤（fsync）來保證資料的持久性。雖然 Kafka 提供了 flush.messages 和 flush.ms 參數來強制 fsync，但官方強烈建議不要使用，因為這會嚴重破壞效能並導致 IOPS 飆升 18。

#### **3.2.2 讀取路徑與預讀（Readahead）**

當消費者讀取資料時，請求首先到達 Page Cache。

* **熱資料：** 對於即時消費（Lag 較小）的場景，資料剛剛被生產者寫入 Page Cache，消費者直接從記憶體讀取，完全不觸及磁碟。  
* **冷資料：** 對於追趕歷史資料的場景，OS 的\*\*預讀（Readahead）\*\*機制會檢測到順序讀取模式，並預先將後續的磁碟塊加載到 Page Cache 中，從而掩蓋磁碟 I/O 的延遲 15。

### **3.3 零拷貝（Zero-Copy）技術與 SSL/TLS 限制**

在傳統的網路傳輸中（如 Java 的 InputStream.read() 到 OutputStream.write()），將檔案資料發送給消費者需要經過 4 次資料拷貝和 4 次上下文切換（Context Switch），消耗大量 CPU 資源 19。

傳統路徑：  
Disk $\\rightarrow$ Kernel Buffer $\\rightarrow$ User Buffer $\\rightarrow$ Kernel Socket Buffer $\\rightarrow$ NIC Buffer  
Kafka Zero-Copy 路徑：  
Kafka 利用 Linux 的 sendfile() 系統呼叫（Java 中對應 FileChannel.transferTo()）：

1. **DMA Copy:** Disk $\\rightarrow$ Kernel Buffer (Page Cache)  
2. **DMA Copy (Scatter/Gather):** Kernel Buffer $\\rightarrow$ NIC Buffer

在這個路徑中，資料**從未**進入使用者空間（User Space），CPU 不參與資料拷貝，上下文切換減少到 2 次。這使得 Kafka 能夠在 CPU 負載極低的情況下飽和網卡頻寬。

SSL/TLS 的限制：  
如果啟用了 SSL/TLS 加密，`Zero-Copy` 機制將會失效。因為加密運算必須在 CPU 上進行，資料必須從核心空間複製到使用者空間進行加密，然後再寫回核心空間發送。這會顯著增加 CPU 的負擔並降低吞吐量，這是在高安全性場景下必須考慮的效能損耗 21。

### **3.4 記憶體映射檔案（mmap）的適用性**

Kafka 對於 .index 和 .timeindex 檔案使用了 **Memory Mapped Files (mmap)** 技術 23。這允許索引檔案被直接映射到虛擬記憶體地址空間，操作索引就像操作記憶體陣列一樣快。

然而，對於主數據檔案（.log），Kafka **不使用** mmap。原因如下：

1. **不可預測的 Page Fault：** 日誌檔案通常很大，隨機寫入 mmap 可能導致頻繁的頁面錯誤，阻塞 I/O 執行緒。  
2. **刷盤控制：** mmap 的 dirty page 刷盤完全由 OS 控制，Kafka 難以控制刷盤時機，這與 Kafka 依賴應用層複製來保證一致性的模型有衝突。  
3. **序列化 I/O 效率：** 對於嚴格順序的追加寫入，標準的 FileChannel 寫入配合 Page Cache 已經足夠高效，且比維護巨大的 mmap 映射更穩定 23。



**4\. 控制平面演進：從 ZooKeeper 到 KRaft**

Kafka 的分散式特性需要一個強大的協調機制來管理 Broker 狀態、Topic 配置、Leader 選舉等元資料（Metadata）。這一層架構正在經歷 Kafka 歷史上最大的變革。

### **4.1 傳統架構：ZooKeeper 依賴與瓶頸**

在 Kafka 2.8 版本之前（以及之後的過渡期），Kafka 強依賴 Apache ZooKeeper 來維護叢集狀態 24。

* **Controller 角色：** 在眾多 Broker 中，有一個會被選舉為 Controller。它負責監聽 ZK 中的節點變化（Watch），並將元資料變更（如 Partition Leader 變更）通過 RPC 廣播給其他 Broker。  
* **架構瓶頸：**  
  * **雙重寫入（Double Writes）：** 元資料既存在於 ZK，也快取於 Controller 記憶體，容易出現狀態不一致。  
  * **恢復時間（RTO）長：** 當 Controller 崩潰時，新 Controller 需要從 ZK 讀取所有 Partition 的元資料。在擁有數十萬 Partition 的大規模叢集中，這個加載過程可能需要數分鐘，期間叢集無法進行 Leader 選舉，處於不可用狀態。  
  * **擴展性限制：** ZK 的寫入能力和 Watch 通知的開銷限制了叢集能支援的分區總數 26。

### **4.2 現代架構：KRaft (Kafka Raft Metadata Mode)**

KIP-500 引入了 KRaft 架構，旨在完全移除 ZooKeeper，實現 Kafka 的自我管理（Self-Managed）26。

#### **4.2.1 內部 Raft 共識機制**

KRaft 在 Kafka 內部實現了一個基於 Raft 共識演算法的控制平面。

* **Quorum Controller：** 叢集中選出一組（通常為 3 或 5 個）Controller 節點。它們之間通過 Raft 協議選舉出一個 Active Controller。  
* **\_\_cluster\_metadata 主題：** 這是 KRaft 架構的核心。所有的元資料變更（如「Topic A 被創建」、「Partition 1 的 Leader 變為 Broker 2」）不再是 ZK 中的 ZNode 變更，而是作為\*\*事件記錄（Event Log）\*\*寫入到一個內部的 Kafka Topic \_\_cluster\_metadata 中。

#### **4.2.2 運作模式的改變**

* **元資料複製：** Active Controller 將元資料變更寫入 \_\_cluster\_metadata 的 Leader 分區。其他 Controller 節點作為 Raft Followers 同步這些日誌。  
* **Broker 作為觀察者：** 普通 Broker 不再被動等待 Controller 的 RPC 推送，而是作為 \_\_cluster\_metadata 的**觀察者（Observer）**，主動從本地或遠端的 Controller 拉取（Fetch）最新的元資料日誌。這是一種事件驅動（Event-Driven）的元資料傳播方式。

#### **4.2.3 KRaft 的優勢**

1. **極速故障恢復：** 由於 Standby Controller 已經通過 Raft 協議即時同步了元資料日誌，當 Active Controller 故障時，新 Leader 的選舉和切換僅需毫秒級，無需漫長的元資料加載過程 24。  
2. **百萬級分區支援：** 移除了 ZK 的瓶頸後，單個 Kafka 叢集可以輕鬆支援數百萬個 Partition。  
3. **統一的運維模型：** 管理員不再需要維護和監控兩套不同的分散式系統（Kafka \+ ZK），降低了運維複雜度。



**5\. 複製機制與資料一致性：ISR, HWM 與 Leader Epoch**

Kafka 的高可用性（High Availability）依賴於其複製（Replication）機制。每個 Partition 有一個 Leader 和多個 Follower。如何在保證效能的同時確保資料不丟失，是這一層的核心難題。

### **5.1 ISR (In-Sync Replicas) 動態集合**

Kafka 並不要求將訊息同步給*所有*副本才算提交，這會嚴重拖慢效能；也不僅僅同步給 Leader 就結束，這會導致資料丟失。它採用了折衷的 ISR 機制 13。

* **定義：** ISR 是一個動態維護的副本集合，包含 Leader 本身以及那些「跟得上」Leader 的 Follower。  
* **判定標準：** 由 replica.lag.time.max.ms 參數控制。如果一個 Follower 在該時間窗口內沒有向 Leader 發送 Fetch 請求，或者發送了請求但沒有追上 Leader 的 Log End Offset (LEO)，它就會被踢出 ISR。  
* **寫入承諾（Committed）：** 當生產者設定 acks=all (或 acks=-1) 時，Leader 必須等到 ISR 中**所有**副本都確認收到訊息後，才向生產者發送 Ack。這保證了只要 ISR 中還有一個節點存活，已提交的資料就不會丟失。

### **5.2 水位線（High Watermark, HWM）與日誌末端（Log End Offset, LEO）**

這是理解 Kafka 資料一致性的核心概念，也是防止「幽靈讀取」與資料不一致的關鍵 29。

| 術語 | 定義 | 作用 |
| :---- | :---- | :---- |
| **LEO (Log End Offset)** | 記錄在日誌檔案末端的下一個寫入位置的 Offset。 | 標示該副本（Leader 或 Follower）物理上寫入到了哪裡。LEO 隨時都在變動。 |
| **HWM (High Watermark)** | ISR 集合中所有副本 LEO 的**最小值**。 | **消費者可見性界限**。Offset \< HWM 的訊息被認為是「已提交」（Committed）的。 |

**詳細同步流程：**

1. **寫入：** Leader 接收訊息，寫入本地 Log，更新自己的 LEO。  
2. **拉取：** Follower 發送 FetchRequest 給 Leader。請求中攜帶了 Follower 自己的 LEO（fetch\_offset）。  
3. **更新 HWM：** Leader 收到所有 ISR 副本的 fetch\_offset 後，計算出最小的 LEO 作為新的 HWM。  
4. **傳播：** Leader 在返回給 Follower 的 FetchResponse 中攜帶新的 HWM。  
5. **截斷：** 消費者只能讀取到 HWM 之前的訊息。這防止了消費者讀取到尚未被所有 ISR 備份的資料（未提交資料）。若 Leader 崩潰，未同步的資料（LEO \> HWM 的部分）可能會被截斷，以保證新 Leader 與其他副本的一致性。

### **5.3 Leader Epoch：解決僵屍 Leader 與資料丟失**

在舊版本 Kafka（0.11 之前），僅依賴 HWM 進行日誌截斷可能導致資料丟失或不一致。例如，一個 Follower 雖然追上了 HWM，但在重啟後可能因為未持久化 HWM 而錯誤地截斷了自己的日誌。為此，Kafka 引入了 **Leader Epoch** 機制 9。

* **定義：** Leader Epoch 是一個二元組 (Epoch, Start\_Offset)。每當新的 Leader 選舉產生，Epoch 版本號加 1，並記錄該 Epoch 開始寫入的第一條訊息的 Offset。  
* **機制：**  
  * 當 Follower 重啟或從故障中恢復並重新加入 ISR 時，它不再盲目根據 HWM 截斷日誌。  
  * 它會發送 OffsetsForLeaderEpoch 請求給當前 Leader，詢問：「在 Epoch X，你的日誌寫到了哪裡？」  
  * Leader 返回該 Epoch 的 End\_Offset。Follower 根據這個權威資訊進行日誌截斷。  
* **解決的問題：** 這有效防止了所謂的「僵屍 Leader」（Zombie Leader）問題，即舊 Leader 復活後可能將未複製的資料暴露給消費者，隨後這些資料又在與新 Leader 同步時發生衝突。Leader Epoch 確保了在任何時刻，副本間的日誌歷史是單調一致的。



**6\. 消費者群組協定與再平衡策略**

Kafka 的消費模型允許多個消費者組成一個 Group，共同消費一個 Topic，實現負載均衡與容錯。這套機制的核心在於**再平衡（Rebalancing）**。

### **6.1 群組協調器（Group Coordinator）**

Broker 端引入了 Group Coordinator 角色來管理消費者狀態。

* **定位：** 透過對 group.id 進行雜湊運算，映射到內部 Topic \_\_consumer\_offsets 的某個 Partition。該 Partition 的 Leader 所在的 Broker 即為該 Group 的 Coordinator 4。  
* **職責：** 接收消費者的 Heartbeat，檢測消費者崩潰，並觸發 Rebalance。

### **6.2 消費者狀態機與加入流程**

消費者加入群組的過程被稱為「後勤之舞」（Logistical Dance）：

1. **FindCoordinator：** 消費者詢問叢集，找到自己的 Coordinator。  
2. **JoinGroup：** 消費者發送加入請求，包含訂閱的 Topic 資訊。Coordinator 選舉第一個加入的消費者為 **Group Leader**（注意：這是客戶端角色，非 Broker 角色）。  
3. **SyncGroup：** Group Leader 根據配置的分配策略（Partition Assignment Strategy），在本地計算出每個成員應該分配到哪些 Partition，然後將分配方案發送給 Coordinator。Coordinator 再將方案分發給所有成員 4。

### **6.3 分區分配策略詳細比較**

Kafka 提供了多種內建策略，影響負載均衡的效果 32：

| 策略 | 邏輯描述 | 優點 | 缺點 |
| :---- | :---- | :---- | :---- |
| **Range Assignor** (預設) | 對每個 Topic 獨立進行範圍劃分。如 10 個分區，2 個消費者，C1 分到 0-4，C2 分到 5-9。 | **Co-location**：如果多個 Topic 分區數相同且 Key 相同，可保證同一 Key 的資料被同一消費者處理，便於 Join 操作。 | 容易導致負載不均（例如 Partition 數量無法被消費者整除時，前面的消費者負擔重）。 |
| **Round Robin** | 將**所有**訂閱 Topic 的 Partition 混在一起，輪詢分配。 | 最大化負載均衡，所有消費者分到的 Partition 數量相差最多為 1。 | 破壞了 Topic 間的 Key 關聯性，導致跨 Topic Join 困難；Rebalance 時變動較大。 |
| **Sticky Assignor** | 盡量維持上一次的分配結果，只將新加入或離開的 Partition 進行最小化移動。 | **穩定性**：大幅減少 Rebalance 期間的分區遷移，保留消費者的本地狀態（快取），適合 Kafka Streams。 | 實作邏輯最複雜。 |

### **6.4 再平衡協定的演進：從 Eager 到 Cooperative**

* **Eager Rebalancing (Stop-the-World)：** 這是傳統模式。一旦觸發 Rebalance，所有消費者必須\*\*立即放棄（Revoke）\*\*當前持有的所有分區，停止消費，等待重新分配完成。這會導致整個群組的消費停頓（STW），在大規模群組中可能導致數秒甚至數分鐘的延遲 35。  
* **Incremental Cooperative Rebalancing (KIP-429)：** 這是現代模式（Kafka 2.4+）。  
  * 消費者不會一次性放棄所有分區。  
  * 在 Rebalance 期間，消費者繼續處理那些**不需要移動**的分區。  
  * 只有那些被判定需要轉移給其他成員的分區才會被暫停和釋放。  
  * 這將一次大的 Rebalance 拆分為兩次小的、幾乎無感知的 Rebalance，徹底消除了全局停頓問題。



## 7\. 事務與精確一次語義（Exactly-Once Semantics, EOS）

Kafka 從早期的「至少一次」（At-Least-Once）進化到「精確一次」（Exactly-Once），依賴於兩個核心機制：冪等性生產者和跨分區事務 37。

### **7.1 冪等性生產者（Idempotent Producer）**

這解決了生產者因為網路超時重試而導致的訊息重複（Duplicates）問題。

* **實作原理：**  
  * 每個生產者在初始化時從 Broker 獲取一個唯一的 **PID (Producer ID)**。  
  * 生產者發送的每批訊息都帶有一個單調遞增的 **Sequence Number**。  
  * Broker 在記憶體中維護每個 \<PID, Partition\> 的最新序列號 N\_last。  
* **去重邏輯：**  
  * 若收到訊息序列號 N \<= N\_last，Broker 判定為重複發送，直接丟棄並返回 Ack。  
  * 若 N \= N\_last \+ 1，接受寫入。  
  * 若 N \> N\_last \+ 1，判定為亂序或中間有丟包，拋出 OutOfOrderSequenceException。

### **7.2 跨分區事務（Transactional Messaging）**

Kafka 支援原子性寫入多個 Partition，這對於 consume-process-produce 模式（如 Kafka Streams）至關重要，確保「讀取資料、處理資料、寫入結果」這三步要麼全部成功，要麼全部失敗。

* **Transaction Coordinator：** 類似 Group Coordinator，負責管理事務的生命週期。  
* `transaction_state`：** 內部 Topic，用於持久化事務日誌（Write-Ahead Log）。這是一個 Keyed Topic，Key 是 `TransactionalId`，Value 記錄了事務的當前狀態。  
* **事務狀態機（Transaction State Machine）：** 事務在生命週期中會經歷以下狀態轉換：

  | 狀態 | 描述 |
  | :---- | :---- |
  | **Empty** | 初始狀態，或事務完成後元數據尚未過期的狀態。此時沒有正在進行的事務。 |
  | **Ongoing** | 生產者已開始事務並向 `Transaction Coordinator` 註冊了至少一個分區。 |
  | **PrepareCommit** | Coordinator 收到提交請求，正在日誌中記錄「準備提交」的意圖；此後將開始向各分區發送 `Commit Marker`。 |
  | **PrepareAbort** | Coordinator 判定事務超時或收到中止請求，正在記錄「準備中止」意圖；此後將開始發送 `Abort Marker`。 |
  | **CompleteCommit** | 所有的 `Commit Marker` 已成功寫入參與分區，事務日誌已更新。 |
  | **CompleteAbort** | 所有的 `Abort Marker` 已成功寫入參與分區，事務日誌已更新。 |
  | **Dead** | `TransactionalId` 超時未更新或被手動刪除，狀態元數據將被清理。 |

* **控制訊息（Control Markers）：** 這是 Kafka 事務的黑科技。  
  * 當事務 `提交 commit` 或 `中止 abort` 時，Coordinator 不會去修改已經寫入 `Partition` 的普通訊息（因為日誌是不可變的）。  
  * 相反，它會向所有參與該事務的 `Partition` 寫入一條特殊的 `Control Marker` 或 `Abort Marker`。
  * 這些 Marker 對普通消費者是不可見的。  
* **隔離級別（Isolation Level）：**  
  * 消費者需配置 `isolation.level=read_committed`。  
  * Broker 或客戶端會緩存未提交的事務訊息。當讀取到 `Commit Marker` 時，才會將該 Marker 之前的資料釋放給應用層；如果讀到 `Abort Marker`，則丟棄對應的事務資料。



## 8\. 流處理中的 Watermark：澄清與對比

在深入 Kafka 原理時，開發者常混淆 Broker 層面的 Watermark 與流處理（Stream Processing）層面的 Watermark。兩者雖然都涉及時間，但作用維度完全不同 40。

| 特性 | Broker High Watermark (HWM) | Event Time Watermark (Kafka Streams/Flink) |
| :---- | :---- | :---- |
| **層次** | 儲存層 / 複製層 | 計算層 / 應用層 |
| **定義** | ISR 副本中最小的 LEO | 一個帶有時間戳 $T$ 的特殊標記或邏輯推斷 |
| **語義** | 代表**資料持久性**。小於 HWM 的資料已備份，不會丟失。 | 代表**事件時間進度**。推斷系統已收到所有時間戳 \< $T$ 的事件。 |
| **處理亂序** | 不處理亂序（Log 是物理有序的）。 | 專門用於處理亂序事件（Late Events）。 |
| **作用** | 決定消費者能讀到哪裡（Consistency）。 | 決定何時觸發視窗計算（Window Triggering）。 |

在 Kafka Streams 中，雖然不像 Flink 那樣顯式發送 Watermark 記錄，但他通過跟蹤每個 Partition 的最大時間戳（Stream Time）來隱式地推進時間，並利用 grace period 來處理亂序資料 43。



## 9\. 結論

Apache Kafka 的架構演進展示了分散式系統設計的極致權衡藝術。它並非單純堆砌功能，而是通過對底層原理的深刻理解來解決問題：

* **I/O 效能：** 通過**順序 I/O** 規避了磁碟尋道開銷，通過 **Page Cache** 與 **Zero-Copy** 規避了記憶體複製與上下文切換，將磁碟 I/O 轉化為接近記憶體操作的效率。  
* **擴展性：** 從 **ZooKeeper** 到 **KRaft** 的轉變，消除了元資料管理的單點瓶頸，開啟了單叢集百萬分區的時代，是架構簡化的典範。  
* **一致性：** 通過 **ISR**、**Leader Epoch** 和 **HWM** 的精巧配合，它在保證高可用（AP）的同時，提供了可配置的強一致性（CP）保障，解決了分散式系統中最棘手的資料丟失與僵屍節點問題。  
* **業務語義：** **冪等性**與**事務**的引入，使其超越了傳統「管道」的角色，成為了可信賴的「事件資料庫」。

掌握這些底層細節——從 Log Segment 的稀疏索引到 Cooperative Rebalancing 的狀態機轉換——不僅是運維 Kafka 的基礎，更是理解現代大規模分散式資料系統設計原理的鑰匙。

## Reference

1. Apache Kafka Architecture Deep Dive \- Confluent Developer, accessed on December 26, 2025, [https://developer.confluent.io/courses/architecture/get-started/](https://developer.confluent.io/courses/architecture/get-started/)  
2. The Engineering Guide to Apache Kafka: Architecture & Internals ..., accessed on December 26, 2025, [https://medium.com/@siddiky/the-engineering-guide-to-apache-kafka-architecture-internals-deep-dive-6f874de4413b](https://medium.com/@siddiky/the-engineering-guide-to-apache-kafka-architecture-internals-deep-dive-6f874de4413b)  
3. Kafka: a Distributed Messaging System for Log Processing \- Notes, accessed on December 26, 2025, [https://notes.stephenholiday.com/Kafka.pdf](https://notes.stephenholiday.com/Kafka.pdf)  
4. Consumer Group Protocol: Scalability and Fault Tolerance, accessed on December 26, 2025, [https://developer.confluent.io/courses/architecture/consumer-group-protocol/](https://developer.confluent.io/courses/architecture/consumer-group-protocol/)  
5. Deep Dive Into Apache Kafka | Storage Internals \- Rohith Sankepally, accessed on December 26, 2025, [https://rohithsankepally.github.io/Kafka-Storage-Internals/](https://rohithsankepally.github.io/Kafka-Storage-Internals/)  
6. Kafka Logs: Concept & How It Works & Format \- AutoMQ, accessed on December 26, 2025, [https://www.automq.com/blog/kafka-logs-concept-how-it-works-format](https://www.automq.com/blog/kafka-logs-concept-how-it-works-format)  
7. Deep dive into Apache Kafka storage internals: segments, rolling ..., accessed on December 26, 2025, [https://strimzi.io/blog/2021/12/17/kafka-segment-retention/](https://strimzi.io/blog/2021/12/17/kafka-segment-retention/)  
8. Understanding Kafka's Internal Storage and Log Retention, accessed on December 26, 2025, [https://conduktor.io/blog/understanding-kafka-s-internal-storage-and-log-retention](https://conduktor.io/blog/understanding-kafka-s-internal-storage-and-log-retention)  
9. Kafka Consumer Offsets Guide—Basic Principles, Insights ..., accessed on December 26, 2025, [https://www.confluent.io/blog/guide-to-consumer-offsets/](https://www.confluent.io/blog/guide-to-consumer-offsets/)  
10. Kafka partition index file \- Stack Overflow, accessed on December 26, 2025, [https://stackoverflow.com/questions/52338890/kafka-partition-index-file](https://stackoverflow.com/questions/52338890/kafka-partition-index-file)  
11. Understanding Kafka Storage Architecture: How It Handles Billions ..., accessed on December 26, 2025, [https://dev.to/ajinkya\_singh\_2c02bd40423/how-kafka-stores-billions-of-messages-the-storage-architecture-nobody-explains-51c6](https://dev.to/ajinkya_singh_2c02bd40423/how-kafka-stores-billions-of-messages-the-storage-architecture-nobody-explains-51c6)  
12. Kafka Log Compaction | Confluent Documentation, accessed on December 26, 2025, [https://docs.confluent.io/kafka/design/log\_compaction.html](https://docs.confluent.io/kafka/design/log_compaction.html)  
13. Kafka, CAP Theorem, and the Zombie Problem \- Medium, accessed on December 26, 2025, [https://medium.com/@wassimsmiti1/kafka-cap-theorem-and-the-zombie-problem-when-log-compaction-challenges-consistency-c15b37af7f28](https://medium.com/@wassimsmiti1/kafka-cap-theorem-and-the-zombie-problem-when-log-compaction-challenges-consistency-c15b37af7f28)  
14. How Kafka Achieves High Throughput: A Breakdown of Its Log ..., accessed on December 26, 2025, [https://dev.to/konstantinas\_mamonas/how-kafka-achieves-high-throughput-a-breakdown-of-its-log-centric-architecture-3i7k](https://dev.to/konstantinas_mamonas/how-kafka-achieves-high-throughput-a-breakdown-of-its-log-centric-architecture-3i7k)  
15. Kafka Design: Page Cache & Performance \- GitHub, accessed on December 26, 2025, [https://github.com/AutoMQ/automq/wiki/Kafka-Design:-Page-Cache-&-Performance](https://github.com/AutoMQ/automq/wiki/Kafka-Design:-Page-Cache-&-Performance)  
16. Design | Apache Kafka, accessed on December 26, 2025, [https://kafka.apache.org/41/design/design/](https://kafka.apache.org/41/design/design/)  
17. Kafka Design Decisions \- by Anadi Gangras \- Medium, accessed on December 26, 2025, [https://medium.com/@anadi.n.gangras/kafka-design-decisions-10c2a423bade](https://medium.com/@anadi.n.gangras/kafka-design-decisions-10c2a423bade)  
18. Hardware and OS \- Apache Kafka, accessed on December 26, 2025, [https://kafka.apache.org/082/operations/hardware-and-os/](https://kafka.apache.org/082/operations/hardware-and-os/)  
19. Kafka's Zero-Copy: The Architecture Behind Lightning-Fast Message ..., accessed on December 26, 2025, [https://krishnakonar12.medium.com/kafkas-zero-copy-the-architecture-behind-lightning-fast-message-delivery-7ffde4aadb7e](https://krishnakonar12.medium.com/kafkas-zero-copy-the-architecture-behind-lightning-fast-message-delivery-7ffde4aadb7e)  
20. What is Zero Copy in Kafka? | NootCode Knowledge Hub, accessed on December 26, 2025, [https://www.nootcode.com/knowledge/en/kafka-zero-copy](https://www.nootcode.com/knowledge/en/kafka-zero-copy)  
21. Zero copy and SSL/TLS : r/apachekafka \- Reddit, accessed on December 26, 2025, [https://www.reddit.com/r/apachekafka/comments/x6i3g5/zero\_copy\_and\_ssltls/](https://www.reddit.com/r/apachekafka/comments/x6i3g5/zero_copy_and_ssltls/)  
22. Implementing mTLS and Securing Apache Kafka at Zendesk, accessed on December 26, 2025, [https://zendesk.engineering/implementing-mtls-and-securing-apache-kafka-at-zendesk-10f309db208d](https://zendesk.engineering/implementing-mtls-and-securing-apache-kafka-at-zendesk-10f309db208d)  
23. why kafka index files use memory mapped files ,but log files don't?, accessed on December 26, 2025, [https://stackoverflow.com/questions/48665257/why-kafka-index-files-use-memory-mapped-files-but-log-files-dont](https://stackoverflow.com/questions/48665257/why-kafka-index-files-use-memory-mapped-files-but-log-files-dont)  
24. Kafka Control Plane: ZooKeeper, KRaft, and Managing Data, accessed on December 26, 2025, [https://developer.confluent.io/courses/architecture/control-plane/](https://developer.confluent.io/courses/architecture/control-plane/)  
25. Comparing KRaft and Zookeeper in Apache Kafka | by anuj pandey, accessed on December 26, 2025, [https://medium.com/@anujp2206/comparing-kraft-and-zookeeper-in-apache-kafka-ddf9ac378eec](https://medium.com/@anujp2206/comparing-kraft-and-zookeeper-in-apache-kafka-ddf9ac378eec)  
26. Apache Kafka® KRaft Abandons the Zoo(Keeper): Part 1, accessed on December 26, 2025, [https://www.instaclustr.com/blog/apache-kafka-kraft-abandons-the-zookeeper-part-1-partitions-and-data-performance/](https://www.instaclustr.com/blog/apache-kafka-kraft-abandons-the-zookeeper-part-1-partitions-and-data-performance/)  
27. KRaft vs ZooKeeper \- Apache Kafka, accessed on December 26, 2025, [https://kafka.apache.org/41/getting-started/zk2kraft/](https://kafka.apache.org/41/getting-started/zk2kraft/)  
28. Why Kafka ditched Zookeeper \- Dev Genius, accessed on December 26, 2025, [https://blog.devgenius.io/why-kafka-ditched-zookeeper-1add2f204d11](https://blog.devgenius.io/why-kafka-ditched-zookeeper-1add2f204d11)  
29. High Water Mark (HWM) in Kafka Offsets \- Level Up Coding, accessed on December 26, 2025, [https://levelup.gitconnected.com/high-water-mark-hwm-in-kafka-offsets-5593025576ac](https://levelup.gitconnected.com/high-water-mark-hwm-in-kafka-offsets-5593025576ac)  
30. difference between Log end offset(LEO) vs High Watermark(HW), accessed on December 26, 2025, [https://stackoverflow.com/questions/39203215/kafka-difference-between-log-end-offsetleo-vs-high-watermarkhw](https://stackoverflow.com/questions/39203215/kafka-difference-between-log-end-offsetleo-vs-high-watermarkhw)  
31. The High Watermark Offset (HWM) in Apache Kafka, accessed on December 26, 2025, [https://blog.2minutestreaming.com/p/kafka-high-watermark-offset](https://blog.2minutestreaming.com/p/kafka-high-watermark-offset)  
32. Chapter 4\. Kafka consumer configuration tuning, accessed on December 26, 2025, [https://docs.redhat.com/en/documentation/red\_hat\_streams\_for\_apache\_kafka/2.7/html/kafka\_configuration\_tuning/con-consumer-config-properties-str](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.7/html/kafka_configuration_tuning/con-consumer-config-properties-str)  
33. understanding Kafka partition assignment strategies \- ️ l-lin, accessed on December 26, 2025, [https://l-lin.github.io/kafka/understanding-Kafka-partition-assignment-strategies](https://l-lin.github.io/kafka/understanding-Kafka-partition-assignment-strategies)  
34. Rebalance & Partition Assignment Strategies in Kafka \- Medium, accessed on December 26, 2025, [https://medium.com/trendyol-tech/rebalance-and-partition-assignment-strategies-for-kafka-consumers-f50573e49609](https://medium.com/trendyol-tech/rebalance-and-partition-assignment-strategies-for-kafka-consumers-f50573e49609)  
35. Rebalance your Apache Kafka® partitions with the next generation ..., accessed on December 26, 2025, [https://www.instaclustr.com/blog/rebalance-your-apache-kafka-partitions-with-the-next-generation-consumer-rebalance-protocol/](https://www.instaclustr.com/blog/rebalance-your-apache-kafka-partitions-with-the-next-generation-consumer-rebalance-protocol/)  
36. Kafka Rebalancing: Concept & Best Practices \- AutoMQ, accessed on December 26, 2025, [https://www.automq.com/blog/kafka-rebalancing-concepts-best-practices](https://www.automq.com/blog/kafka-rebalancing-concepts-best-practices)  
37. Kafka — Internals of Producer and Consumers | by Amit Singh Rathore, accessed on December 26, 2025, [https://medium.com/geekculture/kafka-internals-of-producer-and-consumers-5a1aebb2b3ce](https://medium.com/geekculture/kafka-internals-of-producer-and-consumers-5a1aebb2b3ce)  
38. Kafka Exactly Once Semantics Implementation: Idempotence and ..., accessed on December 26, 2025, [https://www.automq.com/blog/kafka-exactly-once-semantics-implementation-idempotence-and-transactional-messages](https://www.automq.com/blog/kafka-exactly-once-semantics-implementation-idempotence-and-transactional-messages)  
39. KIP-98 \- Exactly Once Delivery and Transactional Messaging, accessed on December 26, 2025, [https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)  
40. Watermarks in Stream Processing Systems \- VLDB Endowment, accessed on December 26, 2025, [http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf](http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf)  
41. Understanding Apache Flink Event Time and Watermarks \- Decodable, accessed on December 26, 2025, [https://www.decodable.co/blog/understanding-apache-flink-event-time-and-watermarks](https://www.decodable.co/blog/understanding-apache-flink-event-time-and-watermarks)  
42. Feature Deep Dive: Watermarking in Apache Spark Structured ..., accessed on December 26, 2025, [https://www.databricks.com/blog/feature-deep-dive-watermarking-apache-spark-structured-streaming](https://www.databricks.com/blog/feature-deep-dive-watermarking-apache-spark-structured-streaming)  
43. Kafka Streams' Take on Watermarks and Triggers | Confluent, accessed on December 26, 2025, [https://www.confluent.io/blog/kafka-streams-take-on-watermarks-and-triggers/](https://www.confluent.io/blog/kafka-streams-take-on-watermarks-and-triggers/)  
44. Why Kafka Streams Does Not Use Watermarks ft. Matthias J. Sax, accessed on December 26, 2025, [https://developer.confluent.io/learn-more/podcasts/why-kafka-streams-does-not-use-watermarks-ft-matthias-j-sax/](https://developer.confluent.io/learn-more/podcasts/why-kafka-streams-does-not-use-watermarks-ft-matthias-j-sax/)