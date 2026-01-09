# MySQL Atomicity & Isolation (Undo Log & MVCC) 的實現方式

Transaction 的 ACID 是寫入資料的重要功能，而 Atomicity & Isolation 則是工程師最常用且最容易影響讀寫效能的兩大特性。

## MySQL Atomicity

直覺思考，可以在資料庫 (記憶體 or 硬碟) 中為每一個 Transaction 建立 private workspace 暫存 Transaction 的異動內容，Commit 時將內容同步到 public workspace 讓其他 Transaction 也看得到，如果 Rollback 就把 private workspace 整個刪除。

然而該方法有兩個致命缺陷：

- private workspace 如果只儲存異動內容，Transaction 重複查詢更新結果，就要不斷從 public workspace 擷取 base 紀錄後加上異動內容，耗效能，如果直接儲存更新後的完整記錄又會導致多個 transaction 開啟多個 private workspace 時佔用太多空間。

- 另外是 Commit 時要同步多筆資料到 public workspace，較耗時，可能造成 Client 送出 Commit 後，在等待中斷線，導致 DB 收到 Commit 執行完回成功，但 Client 沒收到的不一致狀況，Commit 執行越久不一致風險越大。

因此 MySQL 採用相反的設計，Transaction 中所有更新都會同步到 public workspace，也就是 innodb 中的記憶體和硬碟儲存空間，並另外紀錄 Rollback 內容到 Undo Log 空間中，每個紀錄會有一個隱藏的 Rollback Pointer 欄位去指向 Undo Log 內容，Rollback 時將 Undo Log 內容更新到 public workspace，Commit 時去標記 Undo Log 內容為可刪除，隨後由異步 process 刪除 Undo Log。

該方法讓 Commit 變快，Undo Log 只儲存異動內容省空間，且 Transaction 可直接重複查詢更新結果，看似完美了，但還是有兩個問題要解決：

多個 Transaction 同時更新同筆資料，如果 A Transaction 要 Rollback，但 B Transaction Commit 了，A Transaction Rollback 內容會蓋掉 B Transaction 的 Commit，結果就是把 B Transaction 也 Rollback 了，因此多個 Transaction 更新相同紀錄時，必須對該紀錄上鎖，直到 Commit or Rollback 才將鎖釋放。

因為異動都會先同步到 public workspace，其他 Transaction 也會去 public workspace 讀資料，如果被讀到的資料後來被 Rollback 了，相當於其他 Transaction 讀到了髒資料，而這就是 Isolation 要解決的 Dirty Read 問題， Transaction 不能讀到未 Commit 的資料。

## Dirty Read 的解決方案

簡單的方式就是查詢資料時也上鎖，如果別的 Transaction 正在更新資料，就要等 Commit or Rollback 後才能讀到資料，但該方法在併發時效能太差了，有沒有無鎖的機制？

Multiversion Concurrency Control (MVCC) 是 MySQL 實現 Isolation 機制的方法，可在無鎖情況下避免Transaction 讀到未 Commit 資料，其靈感類似 Git 版本控制，當你更新檔案 A 時，其他人可讀取檔案 A 的舊版本，不會看到你沒 commit 的修改。

實作 MVCC 必須儲存不同版本的資料，此時 Undo Log 就派上用場，若當前資料是未 Commit 狀態，透過 Undo Log 裡的 Rollback 內容，就能把資料還原成前一個版本，因為更新時會對資料上鎖，因此同時一筆資料只會有一筆未 Commit 紀錄，因此他的前一版一定是已 Commit 的資料。

但 Isolation 除了要解決 Dirty Read 還要解決 Non-repeatable Read，也就是同一個 Transaction 內讀相同紀錄兩次結果要相同，然而如果該紀錄在兩次查詢之間有被更新並 Commit，就會造成第二次查詢結果與第一次不同。

## Non-repeatable Read 的解決方案

解決 Non-repeatable Read 方法是在 Transaction Begin 時建立資料的 Snapshot，但要把整個資料庫複製一份不現實，可行方式是 Begin 時紀錄資料庫版本號，讀資料發現版本超前時，就透過 Undo Log Rollback 到 Snapshot 版本號，因此 Undo Log 不能只紀錄前一版本的 Rollback，還要維護每筆資料的版本鏈。

Undo Log 中每個 Rollback 節點會有 Pointer 指向前一版本的 Rollback 節點 (如圖)。

![Screenshot 2026-01-05 at 20.08.22](https://hackmd.io/_uploads/r1T5t7tEWg.png)

(圖片來源：https://www.alibabacloud.com/blog/an-in-depth-analysis-of-undo-logs-in-innodb_598966)

此外，每次 Begin Transaction 時 MySQL 會給一個 global 遞增 ID (trx_id)，當 Transaction 更新資料時，會把 trx_id 同步寫到 B+Tree & Undo Log 節點中，因此 trx_id 就是資料庫的版本號。

### 如何決定要 Rollback 到哪個版本？

由於 Transaction 會同時 Begin 多個，無法單純 Rollback Begin 之前的 trx_id，需要透過演算法來決定 Transaction 能讀到哪個版本。

MySQL 在 Transaction Begin 時會建立 Page View 結構：

- m_creator_trx_id：當前 transaction 的 trx_id。
- m_ids：Begin transaction 的當下，所有未 Commit 的 transactions trx_id 集合。
- m_up_limit_id：m_ids 集合中最小的 Trx ID
- m_low_limit_id：下一個 Transaction Begin 時會拿到的 trx_id，也就是未來的版本。並透過下面算法決定哪個版本可讀：id 代表資料當前版本，判斷可否讀規則如下：
    1. 如果 id == m_creator_trx_id 代表是自己修改的紀錄，return true
    2. 如果 id < m_up_limit_id 代表在 Begin 前就 Commit 了，return true
    3. 如果 id ≥ m_low_limit_id 代表在 Begin 後才出現的，return false
    4. 如果 m_up_limit_id ≤ id < m_low_limit_id ，且 id 在 m_ids 裡面，代表在 Begin 當下還沒 commit ，return false
    5. 如果 m_up_limit_id ≤ id < m_low_limit_id，且 id 不在 m_ids 裡面，代表在 Begin 當下已經 commit ，return true
若 return false 就要透過 rollback pointer 往前一版本 rollback，重複檢查直到 return true。

## Transaction 執行太久會怎麼樣？

Transaction 的執行時間對效能是有顯著影響的，若 Transaction 執行很久沒有 Commit/Rollback 會有兩個問題：

1. Begin 之後的 Undo Log 版本持續累積，查詢資料需要 Rollback 很多層
2. 為了要 Rollback 到 Begin 時的版本，Begin 版本後的 Rollback 節點都不能刪除，導致 Undo Log 資料太大，佔用過多記憶體會影響資料快取命中率降低查詢效能。

不過上述問題是在 Isolation Level Repeatable Read (不能 Non-repeatable Read ) 時發生，如果是 Read Committed (不能 Dirty Read) MySQL 是每次查詢都建立 Page View，因此會查詢到後來 Commit 的資料。
