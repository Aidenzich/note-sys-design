> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

# MySQL Atomicity & Isolation (Undo Log & MVCC) 的實現方式

Transaction 的 ACID 是寫入資料的重要功能，而 Atomicity & Isolation 則是工程師最常用且最容易影響讀寫效能的兩大特性。

| 特性 | 說明 |
|------|------|
| **A**tomicity (原子性) | Transaction 要嘛全部成功，要嘛全部失敗回滾 |
| **C**onsistency (一致性) | 資料從一個有效狀態轉換到另一個有效狀態 |
| **I**solation (隔離性) | 並發 Transaction 之間彼此隔離，互不干擾 |
| **D**urability (持久性) | 一旦 Transaction Commit，資料永久保存不會丟失 |



## MySQL Atomicity

要實現 Atomicity（原子性），資料庫必須能夠在 Transaction 失敗時 **完整回滾** 所有已執行的操作。那麼資料庫該如何實現這個機制呢？

**直覺的做法：Private/Public Workspace 模型**

一個直覺的想法是：為每個 Transaction 建立一個 **private workspace**（私有工作區），用來暫存該 Transaction 的所有異動內容。當 Commit 時，再將 private workspace 的內容同步到 **public workspace**（公共工作區），讓其他 Transaction 也能看到這些變更；如果需要 Rollback，就直接把 private workspace 整個刪除即可。

然而該方法有兩個致命缺陷：
- private workspace 如果只儲存異動內容，Transaction 重複查詢更新結果，就要不斷從 Public Workspace 擷取 base 紀錄後加上異動內容，耗效能，如果直接儲存更新後的完整記錄又會導致多個 Transaction 開啟多個 private workspace 時佔用太多空間。
- 另外是 Commit 時要同步多筆資料到 public workspace，較耗時，可能造成 Client 送出 Commit 後，在等待中斷線，導致 DB 收到 Commit 執行完回成功，但 Client 沒收到的不一致狀況，Commit 執行越久不一致風險越大。

因此 MySQL 採用相反的設計，Transaction 所有更新都會同步到 Public Workspace，也就是 innodb 中的記憶體和硬碟儲存空間，並另外<span style="color: orange">紀錄 Rollback 內容到 Undo Log 空間中，每個紀錄會有一個隱藏的 Rollback Pointer 欄位去指向 Undo Log 內容 </span>，Rollback 時將 Undo Log 內容更新到 public workspace，Commit 時去標記 Undo Log 內容為可刪除，隨後由異步 process 刪除 Undo Log。

> **Undo Log 記錄了什麼？**
> 
> Undo Log 記錄的是「如何把資料還原回上一個版本」的資訊：
> 
> | 操作類型 | Undo Log 記錄的內容 |
> |---------|---------------------|
> | **INSERT** | 新插入 row 的 Primary Key（Rollback 時執行 DELETE） |
> | **DELETE** | 被刪除 row 的完整內容（Rollback 時重新 INSERT） |
> | **UPDATE** | 被修改欄位的**舊值**（Rollback 時還原成舊值） |
> 
> 每筆 Undo Log 還包含 `trx_id`（Transaction ID，用於 MVCC）和 `roll_pointer`（指向前一版本，形成版本鏈）。

該方法讓 Commit 變快，Undo Log 只儲存異動內容省空間，且 Transaction 可直接重複查詢更新結果，看似完美了，但還是有兩個問題要解決：

**問題一：多個 Transaction 同時更新同筆資料的衝突**

假設 Transaction A 和 B 同時更新同一筆資料。A 先更新後，B 接著更新，此時 B 的 Undo Log 記錄的是 A 更新後的值。如果 A 要 Rollback，它會把資料還原成 A 更新前的值，但這樣就會<span style="color: orange">覆蓋掉 B 的更新，即使 B 已經 Commit</span>。因此，<span style="color: orange">多個 Transaction 更新相同紀錄時，必須對該紀錄上鎖（Write Lock）</span>，確保同一時間只有一個 Transaction 能修改資料，直到 Commit 或 Rollback 後才釋放鎖。

**問題二：Dirty Read（髒讀）**

因為所有異動都會先同步到 Public Workspace，當 Transaction A 更新資料後還沒 Commit，Transaction B 就可能讀到這筆「尚未確定」的資料。如果 A 後來 Rollback 了，B 讀到的資料就是無效的——這就是 <span style="color: orange">Dirty Read（髒讀）</span>問題。Isolation 的目標之一就是確保 <span style="color: orange">Transaction 不能讀到尚未 Commit 的資料</span>。

## Dirty Read 的解決方案

簡單的方式就是查詢資料時也上鎖（Read Lock），如果別的 Transaction 正在更新資料，就要等 Commit 或 Rollback 後才能讀到資料。但這樣做會導致<span style="color: orange">讀寫互相阻塞，在高併發場景下效能極差</span>。有沒有無鎖的讀取機制？

<span style="color: orange">**Multiversion Concurrency Control (MVCC)**</span> 是 MySQL InnoDB 實現 Isolation 的核心機制，可在**無鎖**情況下避免 Transaction 讀到未 Commit 的資料。

> **MVCC 的核心概念**
> 
> MVCC 的靈感類似 Git 版本控制：當你正在修改檔案 A 時，其他人仍可讀取檔案 A 的舊版本，不會看到你尚未 commit 的修改。

實作 MVCC 必須儲存不同版本的資料，此時 <span style="color: orange">Undo Log 就同時扮演了兩個角色</span>：既用於 Rollback，也用於提供歷史版本給其他 Transaction 讀取。

如果當前資料是未 Commit 狀態，其他 Transaction 可以透過 Undo Log 把資料還原成前一個版本來讀取。由於更新時會對資料上鎖，同一時間一筆資料只會有一筆未 Commit 的紀錄，因此它的前一版一定是已 Commit 的資料。

透過 MVCC，我們解決了 Dirty Read 問題。但 Isolation 還需要解決另一個問題：<span style="color: orange">**Non-repeatable Read（不可重複讀）**</span>。

這個問題是指：在同一個 Transaction 內讀取相同紀錄兩次，結果應該要相同。然而，如果該紀錄在兩次查詢之間被其他 Transaction 更新並 Commit，就會造成第二次查詢結果與第一次不同，破壞了資料的一致性視圖。

## Non-repeatable Read 的解決方案

解決 Non-repeatable Read 的概念是：在 Transaction Begin 時建立資料的 <span style="color: orange">**Snapshot（快照）**</span>，讓該 Transaction 在整個執行期間都只看到這個快照版本的資料。

但要把整個資料庫複製一份顯然不現實。可行的方式是：Begin 時紀錄當前的資料庫版本號，當讀取資料發現版本比快照新時，就透過 Undo Log 往回追溯到快照版本。因此，<span style="color: orange">Undo Log 不能只記錄前一版本，還要維護每筆資料的完整版本鏈</span>，讓我們能回溯到任意歷史版本。

Undo Log 中每個 Rollback 節點會有 Pointer 指向前一版本的 Rollback 節點 (如圖)。

![Screenshot 2026-01-05 at 20.08.22](https://hackmd.io/_uploads/r1T5t7tEWg.png)

(圖片來源：https://www.alibabacloud.com/blog/an-in-depth-analysis-of-undo-logs-in-innodb_598966)

此外，每次 Begin Transaction 時，MySQL 會分配一個<span style="color: orange">全域遞增的 ID（trx_id）</span>。當 Transaction 更新資料時，會把這個 trx_id 同步寫到 B+Tree 的資料頁和 Undo Log 節點中。因此，<span style="color: orange">trx_id 就是版本號</span>，透過比較 trx_id 就能知道資料是在何時被修改的。

### 如何決定要 Rollback 到哪個版本？

由於 Transaction 會同時 Begin 多個，無法單純 Rollback Begin 之前的 trx_id，需要透過演算法來決定 Transaction 能讀到哪個版本。

MySQL 在 Transaction Begin 時會建立 <span style="color: orange">**Read View**</span>（讀取視圖）結構，用來判斷哪些版本的資料對當前 Transaction 是可見的：

| 欄位 | 說明 |
|------|------|
| `m_creator_trx_id` | 當前 Transaction 的 trx_id |
| `m_ids` | Begin 當下，所有**尚未 Commit** 的 Transaction trx_id 集合 |
| `m_up_limit_id` | m_ids 中最小的 trx_id（最早的未完成 Transaction） |
| `m_low_limit_id` | 下一個 Transaction 會拿到的 trx_id（代表未來） |

**版本可見性判斷演算法**

當讀取一筆資料時，設該資料的版本號為 `id`，透過以下規則判斷是否可讀：

```
1. id == m_creator_trx_id → ✅ 可讀（自己修改的）
2. id < m_up_limit_id     → ✅ 可讀（Begin 前就已 Commit）
3. id ≥ m_low_limit_id    → ❌ 不可讀（Begin 後才出現）
4. id 在 m_ids 中         → ❌ 不可讀（Begin 時尚未 Commit）
5. id 不在 m_ids 中       → ✅ 可讀（Begin 時已 Commit）
```

若判斷為不可讀，就透過 `roll_pointer` 往前一版本追溯，重複檢查直到找到可讀的版本。

## Transaction 執行太久會怎麼樣？

Transaction 的執行時間對效能有顯著影響。若 Transaction 執行很久沒有 Commit/Rollback，會造成兩個問題：

1. **查詢效能下降**：該 Transaction 讀取資料時，可能需要沿著 Undo Log 版本鏈往回追溯很多層才能找到可見版本
2. <span style="color: orange">**Undo Log 無法清理**</span>：為了讓長時間執行的 Transaction 能回溯到 Begin 時的版本，這期間所有的 Undo Log 節點都不能刪除，導致 Undo Log 空間持續膨脹，佔用過多記憶體會影響 Buffer Pool 的快取命中率

> [!IMPORTANT]
> 上述問題主要發生在 **Repeatable Read** 隔離級別。在此級別下，Read View 只在 Transaction Begin 時建立一次。
> 
> 若使用 **Read Committed** 級別，MySQL 會在<span style="color: orange">每次查詢時都重新建立 Read View</span>，因此可以看到其他 Transaction 最新 Commit 的資料，Undo Log 也能更快被清理。
