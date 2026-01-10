> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

# MySQL Buffer Pool

資料除了儲存在硬碟中，MySQL 還會將讀取過資料放在記憶體，用來提升查詢效能，但記憶體空間有限，無法把所有資料都放入，因此<span style="color: orange;">淘汰資料的策略就很重要了</span>。

MySQL 不是用 LFU (Least Frequently Used) 策略，因為 LFU 無法適應熱點資料變化，例如客服在某天<span style="color: orange;">頻繁查詢特定用戶的資料，但後續不再使用，在 LFU 的情況下該資料就會一直放在快取中 </span>，且許多業務情境 (e.g 電商 & 交易所 & 搶票) 新建立資料高機率會馬上被查詢，而 <span style="color: orange;">LFU 不適用於這種時間局部性</span>，因為當 LFU 滿了，剛建立資料容易馬上被淘汰。

## Buffer Pool 的 LRU (Least Recently Used)

雖然 LRU 能彌補上述 LFU 缺點，但也會產生新問題，例如執行 `SELECT * FROM users` 全表掃描，依照 LRU 最新載入資料要放進快取並逐出最後讀取資料，因此會把大量資料放進快取並逐出其他資料，然而<span style="color: orange;">全表掃描操作通常為排程任務，有許多非熱門資料，若因此排擠掉熱門資料那就得不償失了</span>。

![Screenshot 2025-12-18 at 20.51.29](https://hackmd.io/_uploads/Byi3Od-mWx.png)

因此 MySQL 用兩個 LRU 結構，分別為 Old & New Sublist，新讀取資料會放到 Old Sublist 若反覆被讀取才放到 New Sublist，在 New Sublist 的資料為長期活躍資料且<span style="color: orange;">不受全表掃描影響(全表掃描產生的大量「一次性資料」只會進入佔比僅 38% 的 Old Sublist，並在該區域快速被汰換。原本存放在 New Sublist（占 62%）的真正熱點資料（如常用用戶資訊、熱門商品）不會被掃描出的垃圾資料擠出記憶體。)</span>，通常 <span style="color: orange;">Old Sublist 佔總快取的 38% 而 New Sublist 為 62%</span>。

**Midpoint Insertion** 的意思是： 當一個新的資料頁（Page）首次從磁碟被讀取到記憶體時，它不會被放在列表的最頂端（也就是最「熱」的位置），而是插入到列表的中間位置（Midpoint），也就是圖中「Old Sublist」的頭部。被系統預設為「尚未證實是熱門資料」。<span style="color: orange;">它必須在 Old Sublist 停留期間被再次存取，才會被搬移到上方 New Sublist 的 Head </span>；否則它會慢慢往下滑，最終被淘汰。


**資料從 Old Sublist 搬移到 New Sublist 的時機為：**

<span style="color: orange;">當資料位於 Old Sublist 後 3/4 段落中，且過 n sec 後再次被讀到才會移到 New Sublist</span>
- n sec 可透過 innodb_old_blocks_time 設定
- 預設 1 sec

等 n 秒設計是因為 MySQL 快取單位為 Page，一個 Page 有多筆連續資料，如果是 Full Table Scan，相同 Page 短時間會被讀取多次，避免 Full Table Scan 搬大量 Page 到 New Sublist，需要等 n 秒設計，而不搬前 1/4 段落是因為這些資料還不會被踢除 ，搬這些資料到 New Sublist 效益比較低。

### InnoDB Buffer Pool 資料查找與載入機制

由於 InnoDB 的快取最小單位為 Page，因此 Buffer Pool 內部的 Hash Map 是透過以下組合來定位資料：

Cache Key 結構： <span style="color: orange;">space_id (Tablespace ID) + page_no (Page Number)。</span>
- `space_id`：定位特定的 Tablespace（表空間）。
- `page_no`：定位該空間內的具體 Page。

- **路徑即成本 (Traversal Path)**： 從 Buffer Pool 查找資料時，必須遵循 B+Tree 結構。這意味著不只是存放資料的 Leaf Page 會被載入，從 Root Page 開始，經過所有 Branch Pages (Non-leaf nodes) 直到 Leaf Page 的完整路徑都會進入 Buffer Pool。因此，<span style="color: orange;">Index 樹的高度越高（層級越多），單次查詢所佔用的 Cache Page 就越多</span>。
- **回表造成的雙重載入 (Clustered & Secondary Index)**： 當執行 `SELECT *`（選取所有欄位）時，若查詢條件使用了 Secondary Index：
    - 系統首先會將相關的 `Secondary Index Pages` 載入快取以找到 Primary Key。
    - 接著為了取得該 Row 的完整資料，必須進行回表 (Back to Table)，再次讀取 Clustered Index Pages。
    - 結論：這種情況下，<span style="color: orange;">`Secondary Index` 與 `Clustered Index` 的 Page 會同時佔用 Buffer Pool 空間</span>。

### 🤨 為什麼 InnoDB 選擇 Page 而非 Record (Row) 作為快取單位？
假設我們將快取單位改為更細粒度的「紀錄 (Row)」，並使用 Hash Map 來管理（Key 為 Space_ID + Primary_Key），雖然看似能節省空間並達到 $O(1)$ 的查詢速度，但會面臨以下致命缺陷：
- 喪失順序性，範圍查詢效能崩潰：Hash Map 天生不具備順序性。當執行 WHERE create_time > ? 這類範圍查詢時，B+Tree 的 Page 結構可以利用 Linked List 快速掃描相鄰資料；但 Row-Level Hash Map 則必須遍歷所有 Key，導致效能極差。
- 空間局部性 (Spatial Locality) 失效：資料存取通常具有群聚效應。例如讀取 id=1 後，極高機率會接著讀取 id=2。
    - **Page Cache**：讀取 `id=1` 時，包含 `id=2` 的整頁資料都已載入快取，後續存取完全不需要 I/O。
    - **Row Cache**：若只快取單筆紀錄，讀取 `id=2` 時可能需要再次發起 I/O 或複雜的查找。註：MySQL 更提供了 Read-Ahead 機制進一步優化此特性。
    - **記憶體管理開銷 (Memory Overhead)**：每個快取單位都需要額外的 Metadata（如 Lock、Pointer、LRU 狀態）。
        - 管理一個 16KB 的 Page 只需要一份 Metadata。
        - 若拆解成 100 筆 Row，則需要 100 份 Metadata。這不僅浪費記憶體，更會大幅增加 CPU 在維護快取結構上的負擔（Parsing & Managing）。
    
- MySQL 提供兩個 read-ahead 功能優化空間局部性
    - **Linear read-ahead**： 如果 extent 中 page 依照硬碟連續空間位置順序被讀取，MySQL 會把物理位置的下一個 extent 透過異步 process 載入到記憶體中，可透過 `innodb_read_ahead_threshold` 參數設定多少個 page 被連續讀取才觸發。
    - **Random read-ahead**： 如果一個 extent 內有許多 page 被隨機讀取，MySQL 會把整個 extent 透過異步 process 載入到記憶體中，可透過 `innodb_random_read_ahead` 開關開啟，官方說如果連續隨機讀取到 13 個 page 才會觸發。

> 注意這裡的物理順序不代表 b+tree 的順序，是硬碟連續空間的意思

### 快取查詢的時間複雜度
大部分情況需要 traversal B+Tree 所以會是 $O(\log N)$，但 MySQL 提供了 Adaptive Hash Table (AHT) 記憶體結構，可透過 `innodb_adaptive_hash_index` 參數開啟，其 key 類似 (table_name, column_name, column_value) 而 value 是指向 Buffer Pool 中儲存資料的 leaf page。
- AHT 大小是動態調整的，會根據查詢情況決定新增或刪除 key，雖然 AHT 能讓快取查詢變成 O(1)，但缺點是高併發下 AHT 效能會很差，因為讀取 AHT 會觸發動態更新，因此多個讀也要競爭相同鎖，若要提高 AHT 併發效能可調整 `innodb_adaptive_hash_index_parts` 會建立多個 AHT partition。
- 而從 Buffer Pool 讀取就是單純讀取，所以可用讀寫鎖，併發讀取效能較好，此外也可調整 `innodb_buffer_pool_instances` 參數建立多個 Buffer Pool partition。

> **相同鎖 (Same Lock / Mutex Contention)**： 指多個執行緒同時競爭同一個互斥鎖（Mutex）。在 CS 意義上，這代表一種「序列化（Serialization）」，也就是所有並行的任務被迫排隊，一次只能有一個人過，這會造成效能瓶頸。
> 
> **讀寫鎖 (Read-Write Lock / RwLock)**： 一種支援「多讀單寫」的機制。它允許多個執行緒同時讀取資料，但若有人要修改資料，則必須獨佔鎖。其核心目標是提高讀取密集型系統的並發性。
>| 特性 | 互斥鎖 (相同鎖) | 讀寫鎖 |
>| :--- | :--- | :--- |
>| **並發性** | 低 (全排隊) | 高 (讀取不互斥) |
>| **適用場景** | 寫入頻繁、邏輯簡單 | 讀多寫少 (如資料庫快取) |
>| **實現複雜度** | 簡單，效能損耗小 | 較複雜，管理鎖狀態需額外 CPU 成本 |



## 優化 MYSQL 的快取命中率

雖然快取能加速查詢，但如果命中率太低幫助就不大，由於 `INSERT` 也會將資料放在快取，查詢剛建立的資料一定會命中快取，因此<span style="color: orange;">理想的快取命中率要在 98% 左右</span>，如果掉到 95% 以下效能就會有明顯下降

執行
```sql
show engine innodb status
```
![alt text](<imgs/Screenshot 2026-01-10 at 7.12.25 PM.png>)

可看到 InnoDB 運作狀況，其中 BUFFER POOL AND MEMORY 段落有 Buffer Pool 相關數據，該段落裡的 Buffer pool hit rate 1000 / 1000 代表一千次查詢中有幾次有命中快取。

當 hit rate 下降時，reads/s (每秒平均有多少查詢需要 I/O) 通常會上升，此時可先檢查 new sublist LRU 中有無資料：

- `Free Buffer` : 可用的空間，單位為 page
- `Database pages` : LRU 全部頁數（不含 free）
- `Old database pages` : old sublist 頁數
- `New Sublist` ≈ Database pages − Old database pages

如果 Free Buffer 很大且 New Sublist Page 不多，代表 Old Sublist 不斷刷新快取，page 來不及進到 New Sublist 就被淘汰了，**此時資料庫大概率在執行全表掃描或大範圍的查詢**。

但如果 New Sublist Page 很多， hit rates 卻下降且 reads/s 升高，可在檢查下面數據：
- `young-making rate` : 1000 次查詢中有幾次在 Old Sublist 命中，且把 Page 移到 New Sublist
- `not young-making rate` : 1000 次查詢中有幾次在 Old Sublist 命中，但沒把 Page 移到 New Sublist

如果 `young-making rate` 低 `not young-making rate` 高，此時查詢只走進 `Old Sublist`，表示 `New Sublist` 有資料但命中率不高，代表查詢模式有變，熱資料需要被更新，但是 `Old Sublist` 卻不斷刷新快取，導致新熱點 page 進不到 New Sublist。

要解決上述問題，<span style="color: orange;">需修復全表掃描，或者調整 `innodb_old_blocks_pct` 參數</span>，把 old sublist 調大，降低 page 被淘汰的機率，提高 page 被移到 new sublist 機率。

### 當 hit rate 下降， young-making rate 卻依然很高時
- Hit Rate 下降：代表 CPU 在快取中找不到資料的頻率變高了，必須頻繁去磁碟讀取 (I/O)。
- Young-making Rate 很高：代表資料在 Old Sublist 待滿預設的 innodb_old_blocks_time (通常 1 秒) 後被成功存取，並被搬移到 New Sublist。

這組合起來的意義是：「資料雖然成功晉升到了熱區 (New Sublist)，但它在熱區待不到足夠長的時間來產生貢獻（提供後續命中），就又因為空間不足被新的資料踢回磁碟了。」
> 📌 這個現象在學術上常被稱為 Cache Thrashing (快取抖動)

此時在升級記憶體空間前，我們可以先檢查 Buffer Pool 都塞了什麼資料。

`information_schema` DB 中的 `innodb_buffer_page_lru` Table 紀錄了快取資料，執行 
```sql
SELECT Table_name, Index_name, Data_size from information_schema.innodb_buffer_page_lru where table_name like "{db_name}.%"
```
可看有哪些 Index 被放進快取中，例如：

![Screenshot 2025-12-22 at 19.38.32](https://hackmd.io/_uploads/r13q6s87Wg.png)

上圖可發現，有三個 Index Tree 在快取，由於 insert 資料時需要更新所有 index 因此會載入多 index tree 的 page 到記憶體中。

然而 Index Name 的 `idx_uid_price` 跟 `idx_uid` 有同等的效果，<span style="color: orange;">因為 `idx_uid_price` 是一個複合索引 (Composite Index)，它的結構是 (user_id, price)。根據最左前綴原則 (Leftmost Prefix Rule)，任何「只查詢 user_id」的請求，都可以直接利用 idx_uid_price 的前半部分來完成。 因此，單獨的 `idx_uid` 索引能做到的事情，`idx_uid_price` 通通都能做到。在功能重疊的情況下，`idx_uid` 就變成了冗餘索引 (Redundant Index)。</span>。

```sql
SELECT * FROM orders WHERE user_id = ? 
```

會命中 `idx_uid_price` 或者 `idx_uid` Index，因此可刪除 idx_uid 減省空間。

此外還可發現 primary Index Tree 的 data size 比開其他 index tree 大 ，因此 <span style="color: orange;">SELECT * 不僅查詢較慢還會載入較多資料到記憶體</span>，而 <span style="color: orange;">index 太多時，也會在 insert 時塞入更多 index tree 到 Buffer Pool 中</span>。

因此可先從優化查詢跟減少 index 數量著手，記憶體還是不夠的話，我們還可看 Buffer Pool 中的 Page Type，執行 

```sql
SELECT DISTINCT (PAGE_TYPE) FROM information_schema.innodb_buffer_page_lru
```

看是否有其他系統資料佔用空間。

![Screenshot 2025-12-22 at 19.41.06](https://hackmd.io/_uploads/ryoE0sI7Wx.png)

查看後，會發現除了 index tree 外還有其他系統資料，其中可能會佔用大量空間的是 Undo Log，Undo Log 是實作 Isolation & Atomicity 功能的結構 (下一章節會說明)，<span style="color: orange;">若 Transaction 執行太久不 Commit，會造成 Undo Log 資料變大</span>，可在 show engine innodb status 的 `Transaction` 段落中查 History list length，該值代表還沒被清除的 Undo Log 數量，若該值很大可透過下面 SQL 查詢執行最久的 Transaction thread ID，然後透過 SHOW PROCESSLIST 找到執行的 Process 並直接 Kill 掉。

## Buffer Pool 數據的理想狀態

- <span style="color: orange;">Free Pages 接近 0 且 Page Types 大多為 Index Page</span>：代表空間有效被利用
- <span style="color: orange;">young-making rate & not young-making rate 長時間接近 0</span>:
代表大部分查詢都有命中 new sublist，查詢模式穩定
- <span style="color: orange;">hit rate 接近 100% 且 reads/s 不多</span>: 代表快取命中率高，I/O 次數不多，查詢普遍都很快
