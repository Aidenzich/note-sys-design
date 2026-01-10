> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

# MySQL 的 Flush Process (快取資料寫入硬碟)

MySQL 寫入流程是：

1. 將資料從硬碟載入 Buffer Pool (記憶體)
2. 在記憶體中更新資料，並寫入異動內容 Log Buffer (Redo Log 記憶體緩衝區) 在寫到 Redo Log File
3. 同時把異動後的 Page (Dirty Page) 記憶體指標放入 Flush List (FIFO Queue)
4. Flush Process 定期從 Flush List 拿出第一個 Dirty Page 並寫入 5. Doublewrite Buffer
5. Doublewrite Buffer 寫入成功後，在寫入 B+Tree 結構的 Page
6. Flush Process 寫入 Dirty Page 標記 Checkpoint LSN，讓 Redo Log 可清除資料

## 那麼什麼是 Log sequence number (LSN)？用途為何？

recover 資料時需要知道從哪一個起點開始重放，因此有 Log Sequence Number (LSN) 用來記錄 Redo Log 寫入狀況，Redo Log 每次新增都會增加 LSN 的數字，是 global 遞增數字，例如某個 `INSERT` 的 LSN 為 100，代表到這次 INSERT 為止，總共寫了 100 bytes 資料到 Redo Log 中。

每個 Page 中會記錄 Oldest Modification LSN 以及 Newest Modification LSN：

Oldest Modification LSN — 該 Page 第一次變 Dirty Page 時的 LSN，Flush List 的順序會從最小的 Oldest Modification LSN 開始刷新到 Disk。

Newest Modification LSN — 該 Page 最後一次被更新的 LSN，可以用來追蹤該 Page 是否短時間內被更新多次，是否需要先被刷新到 Disk。

當 Flush Process 將最小的 Oldest LSN 的 Dirty Page 寫入硬碟時，Checkpoint LSN 就會更新，例如：

- Dirty Page 1: oldest_modification = 900
- Dirty Page 2: oldest_modification = 901，寫入 Dirty Page 1 後，Checkpoint LSN 可往前推進到 901。

而 Checkpoint LSN 可用來當作 recover 資料起點，以及 Redo Log 會清除 LSN 在 Checkpoint 之前的資料。

除了 Checkpoint LSN，InnoDB 也會記錄不同重要時機的 LSN，這些 LSN 可用於確認同步進度並觀測目前寫入的效能，可透過 `show engine innodb status ` 查看：

![Screenshot 2026-01-06 at 20.12.48](https://hackmd.io/_uploads/HJmS2_5V-e.png)

- Log sequence number — 當前最新的 LSN
- Log buffer assigned up to — log buffer 中最新的 LSN 包含未 commit 的 transaction
- Log buffer completed up to — log buffer 中 最新 commit 的 transaction 的LSN
- Log written up to — log buffer 寫入 redo log disk 的 LSN
- Log flushed up to — redo log file 中已經執行 fsync 將資料從 OS cache 更新到硬碟的 LSN
- Added dirty pages up to — flush list 中最晚被更新的 LSN，對應 dirty page 的 newest modification LSN
- Pages flushed up to — flush list 最早被更新的 LSN，對應 dirty page 的 oldest modification LSN
- Last checkpoint at — 最後一筆更新進 B+Tree Disk 的 LSN，Redo Log 會從該 LSN 後進行資料恢復，且 Checkpoint 以前的 Redo Log 內容是可以刪除的，避免 Redo Log 內容太多

當 **Pages flushed up to - Added dirty pages up to** 數值很大時，代表有很多 Dirty Page 等待被 Flush，那麼 Flush Process 觸發的時機為何？如何避免太多 Dirty Page 阻塞？

## Flush Process 觸發的時機

主要兩個觸發點：

- Sharp Checkpoint：將所有 Dirty Page 寫回 Disk，當數據庫正常關閉時觸發。
- Fuzzy Checkpoint：由 Background Cleaner Thread 每秒執行，依照當前 Buffer Pool 狀況動態決定寫回 Disk 的 Dirty Page 數量。

Sharp Checkpoint 可透過 `innodb_fast_shutdown` 參數開啟，但 Fuzzy Checkpoint 的參數就比較複雜了，其有兩種 Flush 模式：

- BUF_FLUSH_LRU — 從 LRU 尾端找出可以 Flush 的 Dirty Pages
為了避免 LRU 驅逐不常用的 Page 時，需要花費額外的時間先 Flush Dirty Page 到硬碟，Cleaner Thread 每秒會依照 innodb_lru_scan_depth 參數從尾端找 N 個 Pages 檢查是否有需要 Flush 的 Dirty Page，BUF_FLUSH_LRU 並不會更新 checkpoint LSN。

- BUF_FLUSH_LIST — 從 Flush List 中最小的 oldest modification lsn 開始 flush，寫入硬碟後會更新 checkpoint LSN

Cleaner Thread 觸發 BUF_FLUSH_LIST 時，不會把所有 Flush List 的 Dirty Page 更新到硬碟，而是透過演算法動態計算出合適的 Dirty Pages 數量，該 Flush 又稱為 Adaptive Flush，其算法為：

`n_pages` 代表幾個 dirty pages 要刷新，他是由三個數字加總取平均值：

1. `(innodb_io_capacity * (ut_max(pct_for_dirty, pct_for_lsn)) / 100` 代表要使用多少全力刷新 dirty pages
innodb_io_capacity 參數代表每秒可有多少 I/O 用於 Flush Dirty Pages，該參數會乘上一個比例決定最終要用多少 I/O，比例由下面兩個數字取大的計算。
`pct_for_dirty` 代表目前 LRU 髒頁的比例 `(modified_pages / total_pages_in_buffer_pool) × 100`，如果小於 `innodb_max_dirty_pages_pct_lwn` 會等於 0，超過 `innodb_max_dirty_pages_pct` 會設定成 100，使用全力 Flush。

2. avg_page_rate 代表平均每秒產生多少 Dirty Pages，計算方式：
`avg_page_rate = ((sum_dirty_pages / time_elasped) + avg_page_rate) / 2`

3. pages_for_lsn Redo LSN 增長了多少，由於 LSN 是 bytes 單位，會將 bytes 單位換算成 Page 數量，計算方式：
total_lsn = current_lsn - checkpoint_lsn;  
avg_lsn_per_page = 2048.0;  
pages_for_lsn = total_lsn / avg_lsn_per_page;

最後三個變數取平均後並確保 page 數量不能超過 `innodb_io_capacity_max` 設定，計算出 n_pages，代表這次要從 flush list 拿出多少個 dirty page 寫進硬碟。

如果發生 Checkpoint LSN 過於落後 Latest LSN，可以透過調大 `innodb_io_capacity` 參數或者調低 `innodb_max_dirty_pages_pct` 參數來增加每次 Flush Dirty Pages 的數量。

如果發現是 Buffer Pool 中 Dirty Pages 比例很高，可以調大 `innodb_lru_scan_depth` 參數，增加 LRU 掃描 dirty page 的範圍，避免淘汰 Page 還要等 Dirty Page 寫入會影響查詢效能。

另外可透過 `show engine innodb status` 的 BUFFER POOL AND MEMORY 查看 Modified db pages數量，該數量代表 Buffer Pool 中有多少 Dirty Pages。

